# Kpopmap Server Architecture  
### Next.js + Spring Boot + WordPress

> 🇰🇷 한국어 / [🇺🇸 English](README_EN.md)
> This repository documents the server architecture strategy of the Kpopmap project.

---

## 🇰🇷 프로젝트 개요

이 저장소는 **Kpopmap 프로젝트의 서버 아키텍처와 설계 철학**을 문서화합니다.

Kpopmap은 단순한 뉴스 소비 사이트가 아니라,  
**합법적으로 수집된 원천 데이터를 기반으로 자체 가공·분석·재구성된 콘텐츠를 제공하는 플랫폼**입니다.

본 아키텍처의 목적은 WordPress를 대체하는 것이 아니라,  
**Kpopmap만의 콘텐츠 파이프라인을 안정적으로 운영하기 위해 역할을 분리하는 것**입니다.

---

## 🇰🇷 콘텐츠 수집 및 저작권 정책

Kpopmap에서 다루는 모든 뉴스·콘텐츠 데이터는 다음 원칙을 따릅니다.

- ❌ 불법 크롤링 없음
- ❌ 무단 복제 / 불펌 없음
- ❌ RSS 무단 수집 없음
- ✅ **공식 뉴스 API 및 허용된 웹 데이터 기반 수집**
- ✅ **수집 이후 Kpopmap 자체 파이프라인을 통해 가공·분석된 2차 콘텐츠**

Kpopmap에 게시되는 콘텐츠는  
**단순 원문 복사가 아닌, 자체 기준과 알고리즘을 거친 Kpopmap 고유 콘텐츠**입니다.

---

## 🇰🇷 프로젝트 배경

Kpopmap은 다음과 같은 특성을 가진 콘텐츠 중심 플랫폼입니다.

- 뉴스 API 및 구조화된 웹 데이터 기반 기사 수집
- 기사 품질 필터링 및 중복/유사도 제어
- AI 기반 분석 및 콘텐츠 재구성
- 읽기 트래픽 중심 서비스
- WordPress 기반 CMS 운영
- Next.js 기반 프론트엔드로 점진적 전환

이 과정에서 WordPress 단일 스택으로는  
**수집·정책·연산·표현을 모두 감당하기 어려운 단계**에 도달했고,  
이를 구조적으로 해결하기 위해 본 아키텍처가 설계되었습니다.

---

## 🇰🇷 설계 원칙

### 1. WordPress는 CMS로 유지한다
WordPress는 다음 역할에 집중합니다.

- 콘텐츠 저장의 기준(Source of Truth)
- 관리자 UI 및 퍼블리싱 워크플로우
- SEO 및 편집 관리

WordPress는 **범용 데이터 처리 서버가 아닙니다.**

---

### 2. Next.js는 사용자 경험을 담당한다
Next.js는 다음을 책임집니다.

- SSR / ISR 기반 렌더링
- UI / UX
- 사용자 요청 처리

공개 콘텐츠에 한해 WordPress REST API를 직접 호출할 수 있습니다.

---

### 3. Spring Boot는 정책 중심 미들웨어이다
Spring Boot는 **중앙 허브가 아닙니다.**

역할은 명확히 제한됩니다.

- 데이터 정책 및 검증 로직
- 내부 API 집계
- 캐시 제어
- Python 워커와 WordPress 간 오케스트레이션

Spring Boot가 중단되더라도  
**프론트엔드와 CMS는 계속 동작해야 합니다.**

---

### 4. Python은 데이터 수집·연산을 담당한다
Python 서비스는 다음 역할을 수행합니다.

- 뉴스 API 및 허용된 웹 데이터 수집
- 기사 본문 정규화 및 품질 필터링
- 중복/유사도 분석
- NLP / AI 기반 전처리 및 분석

Python 서비스는 WordPress에 직접 쓰기 작업을 수행하지 않습니다.

---

## 🇰🇷 아키텍처 다이어그램

