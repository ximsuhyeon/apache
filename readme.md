## 
```
docker run --rm -v "${PWD}:/data" -w /data jitesoft/tesseract-ocr `
  -l eng image.png out
```
# out.txt 생성됨

# Tesseract (jitesoft/tesseract-ocr) Docker 실행 오류 정리 (Windows PowerShell)

아래 내용은 다음과 같은 로그를 기준으로 정리한 해결 가이드입니다.

- `read_params_file: Can't open kor+eng`
- `Warning: Parameter not found: �PNG ...` (PNG 바이너리처럼 깨진 로그가 다량 출력)
- `read_params_file: Can't open out`
- `Error, cannot read input file tesseract: No such file or directory`

---

## 1) 원인 요약: ENTRYPOINT가 이미 `tesseract` 인데 `tesseract`를 또 붙여서 인수 밀림

`jitesoft/tesseract-ocr` 이미지는 보통 **ENTRYPOINT가 `tesseract`로 설정**되어 있습니다.  
그래서 아래처럼 컨테이너 안에서 **`tesseract`를 한 번 더 적어 실행하면**, 실제 인수가 한 칸씩 밀리면서

- 입력 이미지로 `tesseract`(문자열)를 읽으려 함
- `-l` 같은 옵션이 outputbase로 오해됨
- `kor+eng`, `image.png`, `out` 등을 config 파일로 읽으려 함

→ 위 에러/경고가 연쇄적으로 발생합니다.

---

## 2) 해결 1: `tesseract` 단어를 빼고 실행하기 (가장 중요)

먼저 영어로 성공 여부를 확인하세요.

```powershell
docker run --rm -v "${PWD}:/data" -w /data jitesoft/tesseract-ocr `
  -l eng image.png out
# 결과: out.txt 생성
```

> 핵심: **`jitesoft/tesseract-ocr` 뒤에 `tesseract`를 쓰지 않는다.**

---

## 3) 해결 2: `kor+eng` 실패 원인 = 컨테이너에 `kor` 언어 데이터가 없을 가능성

첫 줄의 `read_params_file: Can't open kor+eng` 는 보통 **해당 언어 파일(학습데이터)이 없음**을 의미합니다.

### 3-1) 설치된 언어 확인

```powershell
docker run --rm jitesoft/tesseract-ocr --list-langs
```

여기서 `kor`가 없다면 `-l kor+eng`는 실패합니다.

---

## 4) 해결 3: `kor` 언어팩 포함 커스텀 이미지 만들기 (권장)

한 번만 만들어두면 이후는 편합니다.

### 4-1) Dockerfile

```dockerfile
FROM jitesoft/tesseract-ocr:latest
USER root
RUN train-lang kor --fast
USER tesseract
```

### 4-2) 빌드 / 실행

```powershell
docker build -t my-tesseract-kor .

docker run --rm -v "${PWD}:/data" -w /data my-tesseract-kor `
  -l kor+eng image.png out
# 결과: out.txt
```

---

## 5) (추가) Windows 마운트 폴더 쓰기 권한 문제로 out.txt 생성이 막힐 때

Windows의 특정 폴더(예: `C:\Windows\System32`)나 권한 정책에 따라 컨테이너가 출력 파일을 못 만들 수 있습니다.

### 5-1) 작업 폴더를 “내 폴더”로 옮기기
예: `C:\work\ocr-test` 같은 폴더에서 실행 권장

### 5-2) 컨테이너를 root로 실행

```powershell
docker run --rm --user 0:0 -v "${PWD}:/data" -w /data my-tesseract-kor `
  -l kor+eng image.png out
```

---

## 6) 빠른 점검 체크리스트

1. **`docker run ... jitesoft/tesseract-ocr` 뒤에 `tesseract`를 붙였는가?**  
   - 붙였다면 제거 (ENTRYPOINT 중복)

2. `--list-langs`에 `kor`가 있는가?  
   - 없으면 커스텀 이미지로 `train-lang kor --fast`

3. 출력 파일 생성이 막히는가?  
   - 작업 폴더 위치 변경 또는 `--user 0:0`로 root 실행

---

## 7) 최종 “정상 동작” 예시 모음

### (A) 영어 OCR (기본 성공 확인용)

```powershell
docker run --rm -v "${PWD}:/data" -w /data jitesoft/tesseract-ocr `
  -l eng image.png out
```

### (B) 설치 언어 확인

```powershell
docker run --rm jitesoft/tesseract-ocr --list-langs
```

### (C) 한글 포함 커스텀 이미지 빌드

```powershell
docker build -t my-tesseract-kor .
```

### (D) 한글+영어 OCR

```powershell
docker run --rm -v "${PWD}:/data" -w /data my-tesseract-kor `
  -l kor+eng image.png out
```

### (E) 권한 문제 시 root 실행

```powershell
docker run --rm --user 0:0 -v "${PWD}:/data" -w /data my-tesseract-kor `
  -l kor+eng image.png out
```
