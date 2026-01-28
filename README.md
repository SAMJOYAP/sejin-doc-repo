# sejin-doc-repo

개인 프로젝트 및 학습 내용을 정리하는 **문서 전용 레포지토리**입니다.  
각 프로젝트별로 디렉터리를 분리해, 설계 배경부터 구현 과정, 트러블슈팅까지 기록합니다.

---

## 📁 Projects

### 📦 k8s-toy

Kubernetes 기반 토이 프로젝트 문서입니다.  
로컬 클러스터 구성부터 애플리케이션 배포, CI/CD 자동화까지의 전체 흐름을 다룹니다.

- **Kubernetes Cluster**
  - Kubespray를 활용한 멀티 노드 클러스터 구성
  - MetalLB, Ingress, Metrics Server 등 애드온 설정

- **Application**
  - React 애플리케이션 Docker 이미지 빌드
  - Nginx 기반 정적 파일 서빙
  - Kubernetes Deployment / Service(NodePort) 구성

- **CI / CD**
  - GitHub Actions 기반 파이프라인 구성
  - Self-hosted Runner 운영
  - Docker 이미지 빌드 → Push → Kubernetes 배포 자동화

🔗 관련 레포지토리

- App Repository: https://github.com/SAMJOYAP/sejin-app-repo
- CI/CD 대상 서비스: https://github.com/SAMJOYAP/sejin-app-repo

---

## 🗂️ Repository Structure

```text
sejin-doc-repo
├── k8s-toy/
│   ├── 01-react-dockerize-and-build.md
│   ├── 02-k8s-deployment-and-nodeport-service.md
│   ├── 03-github-actions-cicd.md
│   └── 04-self-hosted-runner-setup.md
└── README.md
```
