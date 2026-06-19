# Hướng Dẫn Kiến Thức Lab 2.2 — Supply Chain Security (Trivy + Cosign)

Tài liệu này giải thích chi tiết 5 kiến thức cốt lõi được áp dụng trong Lab 2.2 cùng với vị trí các đoạn code cụ thể trong dự án thể hiện các kiến thức đó.

---

## 1. Quét Lỗ Hổng Bảo Mật Ảnh Container (Vulnerability Scanning)

* **Khái niệm:** Trước khi phát hành hoặc triển khai ứng dụng, ta phải kiểm tra xem ảnh container (Docker Image) có chứa các thư viện hay phần mềm lỗi thời mang lỗ hổng bảo mật đã biết (**CVE - Common Vulnerabilities and Exposures**) hay không. 
* **Cơ chế hoạt động:** Sử dụng công cụ **Trivy** tích hợp vào CI/CD. Nó sẽ quét toàn bộ hệ điều hành nền (Base OS) và các thư viện ứng dụng bên trong container. Nếu phát hiện lỗ hổng ở mức độ nguy hiểm (**HIGH** hoặc **CRITICAL**), pipeline sẽ trả về lỗi (**exit-code: 1**) để dừng ngay lập tức, ngăn không cho đẩy sản phẩm lỗi lên môi trường chạy.
* **Đoạn code thể hiện:**
  * Tại tệp [.github/workflows/build-push.yml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/.github/workflows/build-push.yml#L65-L73):
    ```yaml
    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@v0.20.0
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
        severity: HIGH,CRITICAL
        exit-code: 1 # Trả về lỗi 1 (Thất bại) để dừng pipeline nếu phát hiện CVE nguy hiểm
      env:
        TRIVY_USERNAME: ${{ github.actor }}
        TRIVY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
    ```

---

## 2. Ký Số Ảnh Container (Container Image Signing - Cosign)

* **Khái niệm:** Nhằm ngăn chặn các cuộc tấn công thay thế ảnh container (tampering/supply chain attack), ta cần xác minh nguồn gốc của ảnh container. Ảnh container chạy trong cụm phải chắc chắn do chính hệ thống CI/CD chính thống của ta xây dựng ra, chứ không phải do hacker tải lên registry.
* **Cơ chế hoạt động:** Sử dụng mã hóa khóa công khai (asymmetric cryptography) thông qua công cụ **Cosign**:
  1. Sử dụng khóa riêng tư (**Private Key** - được bảo vệ và chỉ lưu trong GitHub Secrets) ký mã băm (Digest) của ảnh container.
  2. Đẩy chữ ký lên Registry cùng với ảnh.
* **Đoạn code thể hiện:**
  * Đoạn thực thi ký ảnh trong [.github/workflows/build-push.yml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/.github/workflows/build-push.yml#L75-L83):
    ```yaml
    - name: Install Cosign
      uses: sigstore/cosign-installer@v3.5.0

    - name: Sign the pushed Docker image
      env:
        COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
        COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
      run: |
        cosign sign --yes --key env://COSIGN_PRIVATE_KEY ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
    ```
  * Khóa công khai tương ứng được lưu tại tệp [cosign.pub](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/cosign.pub) dùng làm cơ sở xác thực.

---

## 3. Chính Sách Kiểm Soát Vào Cụm (Admission Control & ClusterImagePolicy)

* **Khái niệm:** Khi Pod được tạo trong Kubernetes, một webhook kiểm tra (**Sigstore Policy Controller**) sẽ chặn lại trước khi Pod được lưu vào cơ sở dữ liệu (etcd). Nó đối chiếu ảnh của Pod với chính sách bảo mật (**ClusterImagePolicy**) để xác thực chữ ký. Nếu ảnh không có chữ ký hợp lệ được ký bởi khóa riêng tư tương ứng, API Server sẽ từ chối tạo Pod.
* **Cơ chế hoạt động:** Tạo tài nguyên `ClusterImagePolicy` trong cluster để cấu hình chữ ký bắt buộc.
* **Đoạn code thể hiện:**
  * Tệp cấu hình chính sách [policies/image-policy.yaml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/policies/image-policy.yaml):
    ```yaml
    apiVersion: policy.sigstore.dev/v1beta1
    kind: ClusterImagePolicy
    metadata:
      name: image-policy
    spec:
      images:
        - glob: "*/**" # Kiểm tra toàn bộ các ảnh từ mọi registry (Docker Hub, GHCR, v.v...)
      authorities:
        - key:
            data: |
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEfKjj98BeRlMJz6MZI6ARFkHd6urn
              vH5JwAVenOTShomIVwjvFWeRWFk152D5HblGVVoKTfpF/TwzfAQ1pDF1vQ==
              -----END PUBLIC KEY-----
    ```
  * Namespace nào muốn áp dụng chính sách này sẽ được kích hoạt bằng nhãn: `policy.sigstore.dev/include=true`.

---

## 4. Loại Trừ Namespace Cho Các Thành Phần Hệ Thống (Namespace Exclusion)

* **Khái niệm:** Nếu ta áp đặt các quy tắc Gatekeeper khắt khe lên toàn cụm một cách mù quáng (ví dụ: bắt buộc mọi Pod phải có giới hạn tài nguyên CPU/RAM, bắt buộc phải có nhãn ứng dụng), các hệ thống quản trị nội bộ của cụm (ArgoCD, External Secrets, OPA Gatekeeper, Rollouts Controller) sẽ bị chính Gatekeeper chặn lại không thể hoạt động hoặc cập nhật.
* **Cơ chế hoạt động:** Cấu hình ngoại lệ loại trừ danh sách các Namespace hệ thống khỏi phạm vi kiểm tra của luật Gatekeeper.
* **Đoạn code thể hiện:**
  * Tại tệp ràng buộc tài nguyên [k8scontainerlimits-constraint.yaml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/opa-gatekeeper/containerlimits/k8scontainerlimits-constraint.yaml#L17-L25):
    ```yaml
        excludedNamespaces:
          - kube-system
          - gatekeeper-system
          - argocd
          - monitoring
          - external-secrets
          - ingress-nginx
          - cosign-system  # Cho phép các Pod của Policy Controller khởi chạy bình thường
          - argo-rollouts   # Cho phép controller argo-rollouts hoạt động điều phối
    ```
  * Tại tệp ràng buộc nhãn [k8srequiredlabels-constraint.yaml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/opa-gatekeeper/requiredlabels/k8srequiredlabels-constraint.yaml#L17-L25):
    ```yaml
        excludedNamespaces:
          - kube-system
          - gatekeeper-system
          - argocd
          - monitoring
          - external-secrets
          - ingress-nginx
          - cosign-system
          - argo-rollouts
    ```

---

## 5. Quản Lý Chính Sách Dưới Dạng Mã Nguồn (GitOps Policy Management)

* **Khái niệm:** Mọi tài nguyên bảo mật hay cấu hình cluster không được tạo thủ công, mà phải được định nghĩa rõ ràng dưới dạng mã nguồn (Declarative YAML) trong kho Git để dễ dàng kiểm soát lịch sử thay đổi (Audit log) và tự động đồng bộ khi có thay đổi.
* **Cơ chế hoạt động:** Sử dụng ArgoCD để trỏ đến thư mục chứa chính sách và tự động đồng bộ.
* **Đoạn code thể hiện:**
  * Khai báo ứng dụng quản lý chính sách trong [argocd/apps/policies.yaml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/temp-mcuong/argocd/apps/policies.yaml):
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: policies
      namespace: argocd
    spec:
      source:
        repoURL: https://github.com/tmcmanhcuong/temp-mcuong.git
        path: policies # Đồng bộ toàn bộ các file trong thư mục policies/ lên cluster
        targetRevision: main
    ```
