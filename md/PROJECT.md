
# AI 분석 시 제외 디렉토리
> **중요**: AI가 프로젝트를 분석/정리/요약할 때 다음 디렉토리들은 읽지 말고 건너뛰어야 함 (잘못된 파일들이라서):
> - `/scripts` - 스크립트 파일들
> - `/docs` - 문서 파일들
> - `/README.md` - 리드미 파일
> - `/SETUP_GUIDE.md` - 문서 파일

# Elasticsearch 클러스터 on Kubernetes 프로젝트

## 프로젝트 개요
이 프로젝트는 AWS EKS 환경에서 Elasticsearch 클러스터를 구축하여 하이브리드 검색(kNN + BM25)과 벡터 데이터베이스 기능을 제공하는 인프라 프로젝트입니다.

### 주요 기능
- **Elasticsearch 8.14.3**: 고성능 검색 엔진
- **하이브리드 검색**: kNN 벡터 검색 + BM25 텍스트 검색 조합
- **벡터 데이터베이스**: 1536차원 dense_vector 지원 (OpenAI 임베딩 호환)
- **Kibana 대시보드**: 데이터 시각화 및 관리
- **자동 스케일링**: HPA 및 Cluster Autoscaler
- **SSL/TLS 보안**: 전체 통신 암호화

## 인프라 스펙
> **기본 운영**: 워크노드 2대 (t3.large)
> **확장 가능**: 트래픽 급증시 최대 4대까지 자동 확장
> **중요**: 평상시 운영에서는 2개 노드 유지 권장

### 최종 확정 리소스 할당 (t3.large 2노드 최적화)
- **Master 노드**: 3개 (각각 512Mi/400Mi 메모리, 500m/100m CPU)
- **Data 노드**: 2개 (각각 3.5Gi/2.5Gi 메모리, 1.2/0.6 CPU)
- **Kibana**: 1개 (2Gi/1Gi 메모리, 1/0.3 CPU)
- **스토리지**: Master 5Gi, Data 40Gi (GP3 SSD)

**참고**: 형식은 `Limit/Request` 순서

## t3.large 2노드 적정성 분석

### ✅ 적정한 이유
1. **리소스 충분성**
   - t3.large: 2vCPU, 8GB RAM per node
   - 총 가용: 4vCPU, 16GB RAM
   - **Request 기준**: 2980m CPU (74.5%), 8.2Gi 메모리 (51%)
   - **Limit 기준**: 4900m CPU (122%), 10.5Gi 메모리 (66%)
   - **여유율**: Request 25.5% CPU, 49% 메모리

2. **성능 특성**
   - Elasticsearch는 메모리 집약적 → t3.large 메모리 충분
   - 검색 워크로드는 CPU 버스트 패턴 → t3 시리즈 적합
   - GP3 스토리지로 I/O 성능 보장

3. **확장성**
   - HPA로 Pod 레벨 스케일링
   - Cluster Autoscaler로 노드 레벨 스케일링
   - 최대 4노드까지 자동 확장

### ⚠️ 주의사항
1. **Request 사용률 모니터링 필요** (현재 74.5% 사용)
2. **CPU 오버커밋 허용** (Limit 122%, Kubernetes 일반적)
3. **스토리지 증설 고려** (벡터 데이터 증가시)
4. **노드 장애시 일시적 성능 저하 가능** (높은 Request 사용률)

## 워크플로우 가이드
향후 이 프로젝트를 수정할 때 참고사항:

1. **리소스 튜닝시**: `manifests/elasticsearch-cluster.yaml`의 resources 섹션 수정
2. **스케일링 정책**: `manifests/elasticsearch-hpa.yaml` 임계값 조정
3. **벡터 검색 최적화**: `hybrid-search-config.yaml` 파라미터 튜닝
4. **모니터링**: Kibana를 통한 클러스터 상태 및 성능 지표 확인

## 배포 구조
```
aws/
├── eks-cluster.yaml          # EKS 클러스터 정의
├── storage-class.yaml        # GP3 스토리지 클래스
└── cluster-autoscaler.yaml   # 클러스터 오토스케일러

manifests/
├── elasticsearch-cluster.yaml # Elasticsearch 클러스터
├── elasticsearch-hpa.yaml     # HPA 설정
├── kibana.yaml               # Kibana 설정
└── elasticsearch-ingress.yaml # ALB 인그레스

config/
├── hybrid-search-config.yaml # 하이브리드 검색 설정
└── vector-db-config.yaml     # 벡터DB 최적화 설정
```

## 리소스 튜닝 최종 정리

### 📊 리소스 사용률 (t3.large 2노드 기준)
```
현재 시스템 (kube-system): 1080m CPU (27%), 2Gi Memory (12.5%)
+ Elasticsearch Master×3: 300m CPU, 1.2Gi Memory
+ Elasticsearch Data×2: 1200m CPU, 5Gi Memory
+ Kibana×1: 300m CPU, 1Gi Memory
────────────────────────────────────────────────
총 Request: 2880m CPU (72%), 9.2Gi Memory (57.5%)
총 Limit: 4900m CPU (122%), 12.5Gi Memory (78%)
```

### 🎯 최적화 전략
1. **Request를 보수적으로 설정**: 안정성 우선
2. **Limit를 충분히 설정**: 버스트 성능 보장
3. **Memory 우선 할당**: Elasticsearch 특성상 중요
4. **CPU 오버커밋 활용**: t3 시리즈 버스트 특성 활용

### 💡 모니터링 포인트
- Request 사용률 80% 이상시 노드 증설 고려
- Memory 사용률 75% 이상시 리소스 증설
- JVM OOM 발생시 힙 메모리 재조정

## 스토리지 설정 세부사항

### GP3 스토리지 클래스 구성
```yaml
# gp3-elasticsearch 스토리지 클래스
parameters:
  type: gp3
  iops: "1000"        # AWS GP3 한계: 볼륨크기 × 500 (5Gi × 500 = 2,500 최대)
  throughput: "125"   # 125 MB/s (보수적 설정)
  encrypted: "true"
```

### IOPS 계산 공식 (AWS GP3)
- **Master 볼륨** (5Gi): 최대 2,500 IOPS → 설정 1,000 IOPS ✅
- **Data 볼륨** (40Gi): 최대 20,000 IOPS → 설정 1,000 IOPS ✅
- **공식**: `max_iops = volume_size_gb × 500`

### ⚠️ 스토리지 트러블슈팅
**문제**: `InvalidParameterValue: Iops to volume size ratio too high`
**해결**:
1. PVC/Pod 삭제 → Storage Class 삭제 → 수정 후 재생성
2. IOPS를 AWS 한계 내로 설정 (볼륨크기 × 500)

### 📊 스토리지 사용량
- **Master**: 5Gi × 3개 = 15Gi (설정/로그용)
- **Data**: 40Gi × 2개 = 80Gi (인덱스 데이터용)
- **총 스토리지**: 95Gi (GP3 SSD, 암호화)

