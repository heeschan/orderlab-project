# OrderLab

단계마다 Ruff와 pytest를 GitHub Actions로 자동 검증

배포 결과: https://heeschan.github.io/orderlab-project/

## 구조

```
orderlab/
  pricing.py           # Level 1-2: subtotal(), final_total()
  csv_report.py        # Level 3: revenue_from_csv()
  api.py               # Level 4: FastAPI (POST /orders, GET /orders/{id})
tests/                 # pytest 테스트
data/sample_orders.csv
generate_report.py     # Level 5: site/index.html 생성
.github/workflows/ci-cd.yml
```

## 실행 방법

```
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements-dev.txt

ruff check .
ruff format --check .
python -m pytest -q
python -m pytest --cov=orderlab --cov-report=xml   # 커버리지(coverage.xml)
python generate_report.py                          # site/index.html 생성
```

## 단계별 구현

| Level | 내용 | 테스트 포인트 |
|---|---|---|
| 1 | `subtotal()` 가격 × 수량 | 정상, 0, 음수(ValueError) |
| 2 | `final_total()` 할인 0~100%, 5만원 미만 배송비 3,000원 | 경계값(49,999 / 50,000 / 50,001), 잘못된 할인율, parametrize |
| 3 | `revenue_from_csv()` | tmp_path, fixture, 빈 파일, 없는 파일, 숫자 오류, 커버리지 artifact |
| 4 | FastAPI 주문 API | TestClient, 201 / 404 / 422 |
| 5 | 정적 보고서 생성과 Pages 배포 | 배포 조건 |

## CI/CD

`.github/workflows/ci-cd.yml`

- `lint`: `ruff check .`, `ruff format --check .`
- `test`: ubuntu와 windows에서 pytest 실행, `coverage.xml`을 artifact로 업로드
- `deploy`: `push`일 때만, 그리고 `lint`와 `test`가 모두 성공한 뒤에만 실행되어 `site/`를 GitHub Pages에 배포

Pull Request에서는 lint와 test만 돌고 deploy는 실행되지 않습니다. 테스트가 실패하면 deploy가 실행되지 않습니다.

## 실패 재현과 해결 기록

- 테스트 기댓값을 바꾼 뒤,`AssertionError`와 함께 expected / actual 차이를 확인하였습니다. 값을 되돌려 통과했습니다.
- 배송비 조건을 `<`에서 `<=`로 바꾸면 경계값 케이스(50,000)가 실패합니다. 테스트로 조건문 내 몇 글자 차이가 결과를 바꾸는 것을 검출합니다.
- 별도 브랜치에서 테스트를 실패시킨 뒤 Pull Request를 만들면, test가 실패하고 deploy는 실행되지 않습니다. Ubuntu가 실패하면 matrix의 나머지 job은 자동 취소됩니다(`fail-fast` 기본 동작). 테스트를 고쳐 다시 push한 뒤 main에 병합하면, deploy가 실행되고 Pages가 갱신됩니다.
- 로컬에서 `httpx` 관련 `StarletteDeprecationWarning`이 보였고, 테스트 결과와는 무관했습니다.
