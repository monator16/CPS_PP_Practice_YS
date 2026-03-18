# Copilot Studio Hands-on 실습 자료 모음

Microsoft Copilot Studio를 활용한 AI 에이전트 개발을 위한 종합 실습 자료입니다. 기본 에이전트 생성부터 멀티 에이전트 구축, 엔터프라이즈 네트워크 통합까지 다양한 시나리오를 다룹니다.

## 실습 구성

### 1. [AgentAcademy](./AgentAcademy/) - 커스텀 에이전트 초급 Workshop

**대상**: Copilot Studio 초중급 학습자  
**출처**: [Microsoft Agent Academy Operative](https://microsoft.github.io/agent-academy)

채용 에이전트를 예제로 사용하여 Copilot Studio의 핵심 기능을 학습합니다.

#### 학습 내용
- **A1. Copilot Studio Introduction** - 기본 인프라 배포 및 에이전트 생성
- **A2. Understanding Agent Models** - 에이전트 모델 커스터마이징 및 최적화
- **A3. Multi Agent** - 단일 에이전트를 멀티 에이전트 시스템으로 전환
- **A4. Integrate with MCP Servers** - MCP(Model Context Protocol) 서버 통합
- **A5. Prompts - Dataverse Grounding** - 엔터프라이즈 데이터 기반 응답 생성

#### 추가 학습 토픽
- 에이전트 지침 작성 및 동작 제어
- 트리거를 통한 에이전트 자동화
- 콘텐츠 조정 및 AI 안전 필수 사항
- 멀티모달 프롬프트로 문서·이미지 처리
- 적응형 카드를 통한 사용자 피드백 수집

**실습 데이터**: `/data` 폴더에 evaluation-criteria.csv, job-roles.csv, resume 샘플 포함

---

### 2. [Agentthon-26-MSKorea](./Agentthon-26-MSKorea/) - 에이전톤 로우코드 데이 실습

**대상**: 멀티 에이전트 협업 시스템 구축  
**작성자**: Solution Engineer 이영서

여러 에이전트가 역할을 분담하여 협업하는 **출장 도우미 멀티 에이전트 Copilot** 실습입니다.

#### 실습 목표
사용자의 출장 관련 질문을 분석하고, 아래 에이전트들이 협업하여 답변:
- **출장 도우미 오케스트레이터** - Intent 분석, Slot Filling, 라우팅
- **출장 규정 전담 에이전트** - 비용 및 규정 관련 질문 처리
- **현지 가이드** - 날씨, 준비물, 생활 정보 제공

#### 학습 내용
- **1. Multi_Agent_Guide.md** - 멀티 에이전트 아키텍처 설계 및 구현
- **2. Trigger.md** - 이벤트 기반 트리거 구성

**샘플 데이터**: `/SampleData` 폴더 포함

---

### 3. [CPS-Vnet-Integration](./CPS-Vnet-Integration/) - VNet 통합 가이드

**대상**: 엔터프라이즈 보안 요구사항을 충족하는 에이전트 구축

Copilot Studio 에이전트를 **Azure 프라이빗 네트워크(VNet)와 안전하게 통합**하는 방법을 다룹니다.

#### 주요 내용
- Power Platform 환경에 Virtual Network 지원 활성화
- Copilot Studio 에이전트와 Azure 리소스(Key Vault, SQL, App Insights 등) 프라이빗 연결
- Public 인터넷을 거치지 않는 안전한 통신 경로 구성
- SQL Server 커넥터 등 VNet 지원 커넥터 활용

#### 장점
- 데이터가 인터넷에 노출되지 않음 → 보안 강화
- 기업 네트워크 정책 준수 가능
- Private DNS 등 네트워크 제어 가능

**참고 리소스**: PowerPlatform-EnterprisePolicies 샘플 코드 포함

---

## 시작하기

### 사전 요구사항
- Microsoft Power Platform 환경 (테넌트)
- Copilot Studio 라이선스
- Azure 구독 (VNet 통합 실습의 경우)

### 학습 순서 추천
1. **AgentAcademy** - 기본기 다지기 (A1 → A2 → A3 → A4)
2. **Agentthon-26-MSKorea** - 실전 멀티 에이전트 구축
3. **CPS-Vnet-Integration** - 엔터프라이즈 레벨 보안 구성

---

## 추가 학습 자료

- [Microsoft Agent Academy](https://microsoft.github.io/agent-academy)
- [Copilot Studio 공식 문서](https://learn.microsoft.com/microsoft-copilot-studio/)
- [Power Platform VNet 통합 가이드](https://learn.microsoft.com/power-platform/admin/vnet-support-setup-configure)
- [MCP (Model Context Protocol) 문서](https://modelcontextprotocol.io/)

---

## 기여 및 피드백

이 자료는 Microsoft의 공식 문서와 실습 자료를 기반으로 구성되었습니다.  
문제가 발생하거나 개선 사항이 있다면 이슈를 생성해 주세요.

---

**Repository**: https://github.com/babycrowLee/Copilot-Studio-Hands-on  
**Last Updated**: March 2026
