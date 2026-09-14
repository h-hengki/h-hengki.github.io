
***
# 회사

## 2026 상반기
### AI Worker : AI를 접하는 단계
* LV3: 매번 하던 반복 업무를 AI가 알아서 정리해주는 단계
* LV4: 간단한 도구로 필요한 AI 기능을 직접 만들어서 쓰는 단계
* LV5: 고급 기법(RAG, CoT)으로 정확도를 높이고 튜닝하는 단계 

## 2026 하반기 
### AIVE : 실제 업무에 적용된 사례를 만드는 단계.
- 단순히 AI 도구를 사용하는 사람이 아니라, AI로 실질적인 가치를 만드는 사람.

***
# 센터

## 2026 상반기 (AI TF STAGE1)

* AI 기본 지식 소개
	* AI 용어소개 : 할루시네이션, MCP, 프롬프트, RAG 등. 인사이트 사이트(ai_insight) 구축
	* 구글 AI 도구소개 : 딥 리서치, Gemini 모드 선택, AI Studio, Stitch, NotebookLM
	* AI 워크플로우 자동화 도구 소개 : Make, n8n, Opal

* AI 페어 프로그램
	* AI 옆에 사수를 붙여서 일하는 방식
	* AI로 프로그래밍 하는 기본 방법을 설명

## 2026 하반기 (AI TF STAGE2)

* 목표 : 하네스 기반 직무 자동화
* 하네스
	* AI = 신입사원
	* 하네스 = 업무 메뉴얼(지침서)
	* 에이전트 = 알아서 일하는 경력사원
* 커리큘럼
	* 1단계 : 클로드 사용법 : 사무업무에 필요한 기능 소개
	* 2단계 : 9단계 실습으로 직접 하네스 구성 방법 설명
	* 3단계 : TF에서 표준 하네스를 구축하고 직원들에게 사용방법 설명


***
# 팀

***
## MES팀

### 1. MES2팀 운영콘솔 I/F 모니터링 화면 개발 및 Oracle DB Function 분석 가이드
  
GMES·BESTERP 두 시스템의 Oracle DB를 한 화면에서 조회·분석할 수 있는 내부 도구.  
  
* ** Interface 상태 확인** : 인터페이스별 실행 이력과 처리 결과를 실시간 조회  
* ** DB 함수 분석 (AI)** : PL/SQL 함수 로직을 AI가 단계별로 트레이스·해설  
* ** TDS 빌더 (AI)** : SQL 실행 결과를 분석해 C# Typed DataSet(.xsd)에 컬럼·설명을 자동 생성 
* SQL 리스트 변환, DB 환경 전환 등 부가 기능 포함  

> 소개 자료 : [[mes2-operconsole.html]]
> 접속 경로 : https://mes-test.seahbesteel.co.kr/mes2


### **2. FlowCore MES - 레거시 화면 암묵지 문서화 도구**  
  
오래된 MES/ERP 화면은 코드만으로는 "왜 이렇게 만들었는지", "어떤 업무 맥락에서 쓰이는지" 파악이 어렵고, 이 지식은 담당자 개인에게만 남아 있는 경우가 많음. 

FlowCore MES는 화면을 선택하면 관련 소스와 DB 구조를 자동으로 수집해 AI가 분석하고, 사람이 읽을 수 있는 결과 문서로 정리해 팀의 자산으로 축적.  
  
프로그램 단위 분석 : 화면 하나의 소스·DB를 근거로 분석 문서 자동 생성  
- 프로세스 분석 : 여러 화면을 업무 흐름(플로우차트)으로 엮어 종합 분석  
- 용어집 : 사내 업무 용어를 팀 공용 사전으로 축적  
- 암묵지 QnA : 쌓인 분석 문서 전체를 근거로 화면을 넘나드는 질의응답  
- 데이터 챗봇(개발중) : BigQuery 원천 데이터에 자연어로 직접 질의**  

> 소개 자료 : ![[FlowCore2_intro.pdf]]
> 기능 매뉴얼 : ![[FlowCore2_Function.pdf]]
> 접속 경로 : http://172.17.40.215:8088



***
## PC팀

### 사내 매뉴얼 RAG 챗봇 구축
- 참여자 : 박광준, 김성훈, 채희민, 이수종
- 주제 : 사내 매뉴얼 RAG 챗봇 구축
- 2026년 KPI 
- 산출물
	- 자연어(평소 말투)로 질문 시 AI가 연관 문서를 즉각 검색 및 답변 제공가능한 쳇봇.
	- 답변에 원본 이미지와 출처 문서를 함께 제시.
	- 매뉴얼을 GitLab에 올리면 자동 반영.
- 소개 자료 
	- https://docs.google.com/presentation/d/1DmJPrnDcFqGFxfOU5D02lNYGtyyIwQwD
- 접속 경로 : http://chat.l2/


### AI를 활용한 Level2 레거시 UI 웹 전환 POC 및 API 파이프라인 구축
- 참여자 : 김태표, 김경원, 서영은, 홍현기
- 주제 : AI 모델링 데이터 파이프라인 구축 및 AI를 활용한 기존 Level2(WPF, WinForm) 화면의 웹 전환 POC
- 2026년 KPI 
- 산출물 
	- 외부 AI 분석 및 내부 데이터 연동을 위한 표준 REST API 서비스 (인증 및 보안 적용)
	- 구축된 API를 기반으로, AI를 활용해 기존 화면(WPF, WinForm)을 웹으로 전환 구현한 'Level2 실시간 조업 모니터링 및 실적 대시보드' (POC)
- 접속 경로 
	- API 문서 : http://172.31.209.71:9080/apidocs/
	- Level2 UI POC 
		- http://app.seahbesteel.co.kr:8004/SMRTWEB002/eaf
		- http://app.seahbesteel.co.kr:8004/SMRTWEB002/eafmon







