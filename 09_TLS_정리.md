# 5주차 핵심 정리 — 전송 계층 보안 (TLS)

> ⚠️ 이 정리는 **28페이지(TLS 1.2 전체 + "Legacy RSA vs ECDHE 개선")까지**만 포함합니다. TLS 1.3 핸드셰이크 상세·QUIC·실습(29p 이후)은 범위에서 제외했습니다.

---

## 1. TLS가 필요한 이유 ⭐

### HTTP 평문 통신의 문제
- HTTP는 요청·응답 **형식**만 정함. 그 자체로 **암호화·서버 인증·변조 검출을 제공하지 않음**.
- 같은 Wi-Fi 공격자가 평문(user, password)을 **읽기·복사·수정** 가능.
- 문제: ① 정상 서버인지 확인 불가 ② 중간 변조 확인 불가.
> 📝 참고: Wi-Fi 비번(WPA2/WPA3)은 **내 기기와 공유기 사이만** 보호. 같은 Wi-Fi 사용자나 공유기 너머 구간에선 HTTP 내용이 보일 수 있음.

### TCP vs TLS 역할 ⭐
| TCP (전송 제어) | TLS (전송 계층 보안) |
|-----------------|---------------------|
| 정확한 전달: 재전송·재정렬·흐름/혼잡 제어 | 안전한 채널 |
| 실수로 깨진 데이터 검사 | **인증**: 누구와 연결했는가 |
| **고의 변조는 못 막음** | **기밀성**: 제3자가 못 읽게 |
| **상대 신원 보증 안 함** | **무결성**: 변경 검출 |
> ⭐ **TCP 연결 성공 = 안전한 상대와 연결 성공이 아니다.**

### 프로토콜 스택에서 TLS 위치
- 응용(HTTP) ↔ **TLS 1.2/1.3** ↔ TCP ↔ IP ↔ 링크. TLS는 응용과 TCP 사이에 넣은 **보호 단계(정식 계층은 아님)**.
- **HTTPS = HTTP over TLS**.
- ⭐ TLS를 써도 **IP 주소·record 길이·시간 정보는 숨기지 않음**.

### TLS의 3대 보안 속성 ⭐
1. **인증(Authentication)**: 인증서 검증 + 개인키 소유 증명으로 서버 확인.
2. **기밀성(Confidentiality)**: 대칭키 암호로 응용 데이터 보호.
3. **무결성(Integrity)**: 인증 tag로 변경 검출(한 비트 변경도 record 거부).

### TLS가 결합하는 암호 primitive ⭐
```
인증서+전자서명(서버 공개키↔이름 연결)
→ Ephemeral key exchange(같은 session secret 계산)
→ KDF(용도·방향별 traffic key)
→ AEAD(데이터 암호화 + 변조 검출)
```
> **Handshake에서 알고리즘·키를 정하고 → Record에서 그 키로 데이터 보호.**

### 버전 역사
- SSL 2.0(1995) → TLS 1.0(1999) → TLS 1.2(2008) → TLS 1.3(2018).
- ⭐ **권고: TLS 1.0/1.1 사용 금지(2021 공식 폐기), TLS 1.2 지원 + TLS 1.3 우선.**

---

## 2. TLS Handshake와 Record ⭐

| **Handshake (연결 설정)** | **Record (데이터 운반)** |
|---------------------------|--------------------------|
| 누구와 어떤 키를 쓸지 | 합의한 키로 무엇을 보호할지 |
| version·알고리즘 제시·선택 | Handshake message를 record에 담아 전달 |
| certificate로 서버 신원 검증 | 응용 데이터를 일정 크기로 분할 |
| ephemeral key share로 공동 secret | 키 확정 후 암호화 + authentication tag |
| **Finished**로 전체 협상 검증 | Alert·연결 종료 신호 |

### Record 형식
- Header(5바이트) = **Content Type(1) + Legacy Version(2) + Length(2)** + Fragment(가변, 키 합의 후 Ciphertext+Tag).
- TLS 1.2: Content Type으로 Handshake·Application Data 구분.
- ⭐ TLS 1.3: ServerHello 이후 **진짜 type을 암호문 안에 숨기고** 겉은 주로 Application Data로 표시.
- 관찰: Wireshark에서 **record length는 보이지만 암호화된 내부는 session key 없이 못 읽음**.

### Message ≠ Record (1:1 아님) ⭐
- 하나의 message(큰 Certificate chain)가 **여러 record로 분할**될 수도, 여러 message(ServerHello+Certificate)가 **한 record**에 들어갈 수도 있음.

### Content Type 값
| 값 | 종류 |
|----|------|
| 22 | Handshake (version·cipher·cert·key 합의) |
| 23 | Application Data (HTTP 등) |
| 21 | Alert (오류·close_notify) |
| 20 | ChangeCipherSpec (TLS 1.2 전환·1.3 호환 신호) |

---

## 3. TLS 1.2 Handshake ⭐⭐ (교수님 "이 그림 중요" 표시)

### 전체 흐름 — **2-RTT** (HTTP 요청 전까지 2번 왕복)
```
① ClientHello                                    ┐ 1번째 왕복
② ServerHello + Certificate                      │ (서버 공개값 수신)
③ ServerKeyExchange + ServerHelloDone            ┘  ②③ 연속 전송
④ ClientKeyExchange                              ┐ 2번째 왕복
⑤ (Client) ChangeCipherSpec, {Finished}          │ (서버 Finished 수신)
⑥ (Server) ChangeCipherSpec, {Finished}          ┘  ④를 받은 뒤 서버가 공유비밀 계산
⑦ {HTTP 요청}  ← 총 2 RTT 뒤 전송
```
- RTT = Round-Trip Time(왕복 시간). RTT 50ms면 HTTP 전 TLS 대기 ≈ **100ms**(2 RTT).
- `{ }`는 암호화. 📝 필기(정확): **서버에 대한 인증 O, 클라이언트에 대한 인증 X**(일반 서버 인증). 키 교환은 **DH-key 교환(ECDHE)** 사용, 과거엔 RSA도.

### ① ClientHello ⭐
- client_random 생성, 지원 목록 제안. 서버가 공통 version·암호 있는지 확인.
- 포함: 지원 암호 조합(cipher_suites), 서명 알고리즘, 타원곡선 그룹, **SNI**(접속할 서버 이름).
- ⭐ 아직 비밀번호 안 보냄. **이 단계는 평문이라 SNI도 보임**. 최종 선택은 서버가.

**ClientHello 필드**: client_version(TLS 1.2), random(32바이트), cipher_suites, extensions(SNI·signature algorithms·groups).
- **SNI (Server Name Indication)**: 한 IP에서 여러 HTTPS 사이트 운영 시 접속 hostname을 알려줌.

### Cipher Suite 읽기 (TLS 1.2) ⭐
`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
| 부분 | 의미 |
|------|------|
| **ECDHE** | 일회성 key agreement (키 교환) |
| **RSA** | 서버 인증용 RSA 서명 (인증) |
| **AES_128_GCM** | record 암호화 + 변조 검출 (AEAD) |
| **SHA256** | key derivation·transcript hash |

### ② ServerHello + Certificate
- ServerHello: version·suite 선택, server_random 생성. Certificate: 서버+중간 인증서 전달.
- ⭐ Certificate는 **이름↔공개키 연결** 제공. **인증서 전달만으로 개인키 소유 증명 안 됨** → 다음 서명 검증 필요.

**인증서 검증 항목** (3주차와 동일): ① 이름(SAN에 hostname) ② 기간(notBefore~notAfter) ③ 서명(chain) ④ 신뢰(root가 trust store에) ⑤ 상태(폐기).

### ③·④ KeyExchange ⭐ (ECDHE 복습 연결)
- **ECDHE 복습**: 같은 기준점 G, 매 핸드셰이크마다 임시 a·b 생성 → A=aG, B=bG 공개값 교환 → `S = aB = bA = abG` (공유점). **a·b·S·Z는 전송 안 함**.
  - P-256: `Z = encode32(x(S)) = pre_master_secret (32바이트)`. X25519도 32바이트 Z.
- **흐름**: ServerKeyExchange(B + **RSA 서명**) → ServerHelloDone → ClientKeyExchange(A).
  - RSA 서명 입력: `R_C ‖ R_S ‖ params`(곡선 정보·B). R_C=ClientHello.random, R_S=ServerHello.random (각 32바이트 공개 난수).
  - ⭐ **RSA는 B(서버 공개값)를 인증**한다. **Z를 암호화·전송하지 않음** (Z = pre_master_secret).
  - 서버는 ④(A) 받은 뒤 S=bA, Z 계산. 클라는 B 받은 뒤 S=aB, Z 계산.

### 키 유도 (MS, KB) — 같은 비밀에서 방향별 키·IV ⭐
- `PRF₂₅₆(secret, label, seed, L)`: HMAC-SHA256 기반 의사난수 함수.
- **MS (master secret, 48B)** = `PRF₂₅₆(Z, "master secret", R_C‖R_S, 48)`.
  - EMS(Extended Master Secret) 사용 시: 핸드셰이크 transcript hash를 seed로.
- **KB (key_block, 40B)** = `PRF₂₅₆(MS, "key expansion", R_S‖R_C, 40)`. (주의: 난수 순서가 **R_S‖R_C**로 바뀜)
- 방향별 분리:
  | 방향 | AES 키(16B) | 고정 IV(4B) |
  |------|-------------|-------------|
  | C→S | KB[0:16] | KB[32:36] |
  | S→C | KB[16:32] | KB[36:40] |
- GCM은 별도 MAC 키 안 만듦.
> 📝 필기(정확): "client, server만 안다(ECDH). 유출되면 X." 즉 Z는 양측만 계산 가능.

### ⑤·⑥ Finished — 같은 키·협상 기록 검증 ⭐
- `V_C = PRF₂₅₆(MS, "client finished", SHA256(transcript_C), 12)`, 서버는 "server finished".
- 양측이 서로의 Finished를 복호·검증. **검증값 불일치 시 중단** → 중간에서 협상이 조작되지 않았음을 확인.
> 📝 필기: "없어도 큰 문제 X"라고 적혀있으나 — **교정**: Finished는 **다운그레이드·변조 공격 탐지**에 중요. 전체 handshake transcript를 검증하므로 생략하면 협상 무결성이 깨짐. (교수님 코멘트는 "로그인 자체 동작엔 영향 적다"는 맥락으로 이해, 보안상으론 필수)

### TLS 1.2 가시성 (어디부터 암호화?) ⭐
| 메시지 | 패킷에서 보이는 것 |
|--------|---------------------|
| ClientHello | 평문: 제안 목록·난수 |
| ServerHello/Certificate | 평문: 선택값·**인증서** |
| KeyExchange | 평문: 공개값·서버 서명 |
| ChangeCipherSpec | 평문: 암호 전환 신호 |
| **Finished / HTTP** | **암호문**: 키 로그 없이 내용 불가 |
> ⭐ **각 측의 Finished부터 암호화.** 서버 인증서가 평문이어도 개인키는 노출 안 됨.

### Record 보호 (AES-GCM) ⭐
- `(Cᵢ, Tᵢ) = AES-128-GCM-Encrypt(K_C→S, Nᵢ, Pᵢ, AADᵢ)`.
- **AAD(13바이트)** = seq ‖ type ‖ version ‖ length → 암호화 안 하고 **인증에 포함**.
- Nonce Nᵢ = IV_C→S(4B) ‖ nonce_explicit(8B) = 12B. nonce_explicit는 각 record에 전송.
- ⭐ **같은 키에서 전체 nonce 재사용 금지** (2주차 GCM 원칙과 동일). S→C는 K_S→C, IV_S→C 사용.

---

## 4. Legacy RSA key exchange vs ECDHE ⭐⭐ (28p, 범위의 마지막)

| **과거 · RSA key exchange** | **현재 · ECDHE** |
|------------------------------|------------------|
| 서버 public key로 pre-master secret **암호화** | certificate key는 ephemeral share **서명**에만 사용 |
| 공격자가 과거 TLS traffic 저장 | certificate private key는 **인증만** 담당 |
| 나중에 서버 certificate private key 탈취 | shared secret은 임시 a·b에서 생성 |
| 저장한 encrypted pre-master secret 복호 | 나중에 장기 private key가 탈취돼도 |
| → **과거 session까지 복호 가능** ❌ | → a·b가 이미 삭제돼 **과거 통신 안전** ⭕ (Forward Secrecy) |

> ⭐⭐ **핵심: TLS 1.3은 RSA를 없앤 게 아니라 '정적(static) RSA key exchange'를 제거했다.**
> - RSA는 TLS 1.3에서도 **인증서와 RSASSA-PSS 서명**에 여전히 사용 가능.
> - 제거된 것은 **'장기 RSA key로 premaster secret을 복호하는 key transport'** 방식뿐.
> - 이유: static RSA는 **Forward Secrecy가 없기** 때문. ECDHE는 임시키로 FS 제공.
> 📝 필기(정확): "성명윤" 표시 — 이 슬라이드 강조. static RSA 제거가 핵심.

---

## ✅ 핵심 정리 (28p까지)
1. **TLS가 필요한 이유**: HTTP 평문은 인증·기밀성·무결성 없음. TCP 연결 성공 ≠ 안전한 상대.
2. **Handshake vs Record**: 전자는 알고리즘·키 합의, 후자는 그 키로 데이터 보호. message:record는 1:1 아님.
3. **TLS 1.2 = 2-RTT**: ClientHello→ServerHello/Cert→KeyExchange→Finished. 각 측 Finished부터 암호화.
4. **Cipher suite**: 키교환(ECDHE)·인증(RSA)·AEAD(AES-GCM)·해시(SHA256) 4부분.
5. **키 유도**: Z → MS → KB → 방향별 키·IV (PRF 기반, 양측이 동일 계산).
6. **ECDHE가 static RSA를 대체한 이유 = Forward Secrecy**. TLS 1.3은 static RSA key exchange만 제거(인증서 RSA는 유지).
