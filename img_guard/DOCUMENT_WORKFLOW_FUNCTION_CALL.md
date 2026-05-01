# 문서 확장 함수 호출 가이드

작성 기준: Django 백엔드에서 AI 모듈을 직접 import해 호출하는 구조

## 범위

초기 문서 MVP는 `표준 근로계약서`를 대상으로 한다.

목표는 OCR 기반 자동 위변조 확정 판정이 아니라 다음 네 가지다.

- 문서 원본/워터마크본 보관
- 문서 페이지 워터마크 삽입/검출
- 기존 이미지와 동일한 토큰 발급 플로우 연계
- OCR 최소 요약 필드 추출

OCR 요약 필드는 토큰에 넣지 않는다. 토큰에는 기존 이미지와 동일 필드만 사용한다.

## 상태 정의

문서 워크플로우는 이미지의 `allow/review/block`과 다르게 처리 신뢰도 기준을 사용한다.

- `verified`: 문서 처리 성공. 워터마크 삽입/검출이 성공했고, 등록의 경우 OCR 최소 필드가 일정 수준 추출됨
- `review`: 워터마크는 처리됐지만 OCR 필드가 부족하거나 검증 시 워터마크 검출이 실패해 확인 필요
- `failed`: 파일 처리, 렌더링, 워터마크, OCR 등 기술 처리 실패

`verified`는 법적 진위 확정이 아니다. 서비스 내 인증 처리 완료 상태다.

## 등록 함수

```python
from app.document.workflow_service import run_document_register_workflow_v1

resp = run_document_register_workflow_v1(request_dict)
result = resp.model_dump()
```

최소 요청:

```python
request_dict = {
    "job_id": "uuid",
    "input": {
        "s3_key": "document/original/u1/c1/contract.pdf",
        "filename": "contract.pdf",
        "mime_type": "application/pdf",
    },
    "meta": {
        "user_id": "u1",
        "content_id": "c1",
    },
    "document_type": "labor_contract_std_v1",
}
```

응답 주요 필드:

```python
{
    "success": True,
    "decision": "verified" | "review" | "failed",
    "assets": {
        "original_s3_key": "...",
        "watermarked_s3_key": "...",
        "ocr_raw_s3_key": "...",
    },
    "watermark": {
        "applied": True,
        "payload_id": "...",
        "output_key": "...",
    },
    "ocr_summary": {
        "representative_name": {"value": "..."},
        "worker_name": {"value": "..."},
        "written_date": {"value": "..."},
        "missing_fields": [],
    },
    "pending_actions": [
        "mint_token_with_existing_image_fields",
        "save_document_ocr_summary_to_db",
    ],
}
```

백엔드는 `pending_actions`에 따라 기존 이미지와 동일한 토큰 발급 로직을 호출하면 된다.

## 검증 함수

```python
from app.document.workflow_service import run_document_verify_workflow_v1

resp = run_document_verify_workflow_v1(request_dict)
result = resp.model_dump()
```

최소 요청:

```python
request_dict = {
    "job_id": "uuid",
    "input": {
        "s3_key": "document/verify/u1/request.pdf",
        "filename": "request.pdf",
        "mime_type": "application/pdf",
    },
    "meta": {
        "user_id": "u1",
    },
}
```

검증 응답에서 `watermark.payload_id`가 있으면 백엔드가 해당 payload/token 매핑을 조회한다.

## OCR 최소 필드

현재 저장 권장 필드는 다음 3개다.

- `representative_name`
- `worker_name`
- `written_date`

이 값은 DB 저장용 보조 정보다. 온체인 토큰 필드로 사용하지 않는다.

## 운영 환경변수

기존 이미지 환경변수에 아래 값을 추가한다.

```bash
CLOVA_OCR_INVOKE_URL=...
CLOVA_OCR_SECRET=...

DOC_DEFAULT_TYPE=labor_contract_std_v1
DOC_RENDER_DPI=220
DOC_MAX_PAGES=5
DOC_OCR_TIMEOUT_SEC=30

S3_PREFIX_DOC_REGISTER_REQUEST=document/register_request
S3_PREFIX_DOC_VERIFY_REQUEST=document/verify_request
S3_PREFIX_DOC_WATERMARK_RESULT=document/watermarked
S3_PREFIX_DOC_OCR_RAW=document/ocr_raw
S3_PREFIX_DOC_PREVIEW=document/preview
```

DOCX 지원에는 서버에 `libreoffice`가 필요하다. PDF만 우선 지원하면 `libreoffice` 없이도 동작한다.

PDF 렌더링에는 Python 패키지 `pymupdf`가 필요하다.

## 백엔드 저장 권장

문서 등록 테이블 또는 기존 content 확장 필드에 다음을 저장한다.

- 원본 S3 key
- 워터마크본 S3 key
- OCR raw JSON S3 key
- OCR 요약 필드 3개
- OCR 상태(`verified/review`)
- 워터마크 payload_id
- token_id / tx_hash

## 주의

- OCR 추출값은 검색/조회/확인 보조 정보다.
- OCR 누락만으로 자동 차단하지 않는다.
- 계약서 내용 위변조 확정 판정은 MVP 범위에서 제외한다.
- 필요 시 보관 원본 또는 워터마크본을 기준으로 사람이 재확인한다.
