# 3주차 핵심 정리 ② — 인증서·PQC 전환 + 해시에서 인증까지

---

# PART A. 인증서와 양자내성 전환 (공개키 자료 후반부)

## 1. 인증서 (Certificate) ⭐
- **왜?**: 공격자가 자신의 public key를 **서버의 key라고 바꿔치기**하지 못하게. identity와 public key를 묶고 **신뢰하는 CA가 서명**.

### X.509 인증서 필드
- **정의**: 공개키 + 신원·유효기간·사용목적을 담고 발급자가 전자서명한 표준 데이터 구조.
- **인코딩**: ASN.1로 구조 정의 → **DER**로 byte를 유일하게 표현.
- **주요 필드**: Subject(www.shop.example), Issuer(발급 CA), Validity(유효기간), **SPKI**(공개키), **SAN**(DNS 이름들), **Key Usage/EKU**(digitalSignature/serverAuth), Serial, 서명 알고리즘.
- **왜 이름·시간·용도를 서명에 포함?**: 정상 key가 **다른 사이트·다른 목적으로 오용**되는 것을 막기 위해.
> 📝 필기(정확): "X.509 = ISO 표준, 실제 구현·이론에서 널리 활용."

### 인증서 파일 형식 (저장·전송 목적별)
| 형식 | 내용 | 확장자 |
|------|------|--------|
| **DER** | ASN.1을 binary로 유일 인코딩 | .der .cer |
| **PEM** | DER를 Base64 text + 머리말 | .pem .crt (-----BEGIN CERTIFICATE-----) |
| **PKCS#12/PFX** | 인증서+**개인키**+chain 묶음(암호화 가능) | .p12 .pfx (백업·이전) |
| **PKCS#7/CMS** | 인증서+chain (보통 개인키 미포함) | .p7b .p7c (중간 인증서 묶음) |

## 2. 신뢰 저장소 (Trust Store) ⭐
- **정의**: OS·브라우저·조직이 **trust anchor**로 인정한 **Root CA 인증서 모음**.
- 검증: leaf → intermediate CA → root CA가 **trust store에 있는가?** YES여야 신뢰 후보.
- ⭐ **공격자 root가 등록되면**: 공격자가 임의 사이트용 가짜 인증서 발급 가능 → MITM을 정상 연결처럼 수용. (기업 TLS inspection도 같은 원리로 조직 root 설치 → root 추가는 관리자급 보안 결정)
- Windows: certmgr.msc / `Get-ChildItem Cert:\LocalMachine\Root`. Chrome: Chrome Root Program, Firefox: Mozilla Root Store.

## 3. CA 계층과 경로 검증
- **Root CA**는 서버 인증서를 직접 대량 발급하지 않고 **Intermediate CA에 위임**. Root는 self-signed·offline 보호.
- 서버는 보통 **leaf + intermediate**를 전송(Root는 client trust store에 이미 있어 안 보냄).

### 인증서 경로 검증 (서명 하나가 아님) ⭐
1. **경로 구성**: leaf → intermediate → trust store의 root 연결
2. **서명 검증**: 각 인증서를 상위 발급자 공개키로 검증
3. **시간 검증**: notBefore ≤ 현재 ≤ notAfter
4. **이름 검증**: 접속 hostname이 **SAN**에 있는가
5. **용도·제약·상태**: Key Usage, EKU, CA 제약, revocation, 정책
> 실패 예: shop.example 인증서를 bank.example에 제시 → 이름 불일치. unknown root → 신뢰 실패.

## 4. 인증서 폐기 (Revocation) ⭐
- 유효기간이 남아도 **private key 유출 시 즉시 폐기** 필요.
| 방식 | 동작 |
|------|------|
| **CRL** (Certificate Revocation List) | CA가 폐기 인증서 serial 목록을 서명해 주기 배포 (📝 "오프라인 방식") |
| **OCSP** (Online Certificate Status Protocol) | client가 특정 인증서 상태를 responder에 질의 → good/revoked/unknown (📝 "온라인·실시간") |
| **OCSP stapling** | 서버가 서명된 OCSP 응답을 미리 받아 **handshake에 첨부** → client 별도 질의·privacy·지연 감소 |
> 주의: 네트워크 실패 시 차단/허용 정책이 달라 revocation은 만능이 아님.

### CA 사고 사례 (Entrust 2024)
- 반복된 compliance 실패·개선 미이행 누적 → 2024-06 Chrome이 Entrust 신규 TLS 인증서 **기본 신뢰 중단** 발표, 2024-11 distrust 시행.
- ⭐ 교훈: "인증서 한 장 미폐기로 즉시 해킹"이 아니라, **반복된 사고·규정 대응 실패 누적 → CA root 전체의 신뢰 상실**.

## 5. Forward Secrecy (FS) ⭐⭐
- **정의**: 장기 인증키가 **나중에 유출돼도** 과거 세션 평문을 복원 못 하게.
- 시나리오: ① 오늘 암호화 트래픽 녹화 → ② 몇 달 뒤 서버 장기 private key 탈취 → ③ 과거 세션키 재계산 가능한가?
  - **FS 없는 key transport**(정적 RSA): 장기키로 과거 premaster/session secret 복구 → 녹화 통신 노출 ❌
  - **FS 있는 ephemeral**: 임시 a·b를 사용 후 폐기 → 장기키만으로 과거 Z·세션키 재현 불가 ⭕
- ⭐ 성립 조건: ephemeral key를 안전 생성 + 세션 후 **삭제**. **현재 endpoint 감염까지 막는 건 아님**(과거 피해 범위 제한).

## 6. 양자내성암호 (PQC)
| 계열 | 양자 영향 | 대응 |
|------|-----------|------|
| RSA·DH·ECC | **Shor** 알고리즘이 기반 난제 위협 | PQC KEM/signature로 전환 |
| 대칭키 | **Grover**로 brute force 제곱근 가속 | 충분한 key 길이 margin |
| Hash | 양자 분석 | 출력 길이·강도 평가 |
| 장기 비밀 | **harvest now, decrypt later** | 지금부터 inventory·migration |

### NIST 2024 FIPS PQC 표준 (KEM 1 + 서명 2) ⭐
- **FIPS 203 = ML-KEM** (Module-Lattice KEM, CRYSTALS-Kyber 유래)
- **FIPS 204 = ML-DSA** (Module-Lattice 서명, CRYSTALS-Dilithium 유래)
- **FIPS 205 = SLH-DSA** (Stateless Hash 서명, SPHINCS+ 유래)
- **하이브리드 전환**: Inventory→Agility(versioning)→**Hybrid(classical+PQ 결합)**→Observe→Retire. "둘 중 하나만 안전해도 전체 안전"하도록 결합 + downgrade 방어.

### 2026 알고리즘 선택 가이드 ⭐
| 필요 | 우선 | 피할 것 |
|------|------|---------|
| 데이터 AEAD | AES-GCM / ChaCha20-Poly1305 | ECB, unauthenticated CBC/CTR |
| 일반 hash | SHA-256/384, SHA-3 | MD5·SHA-1 신규 |
| Password | Argon2id (규제 시 승인 KDF) | 단일 SHA-256·고정 salt |
| Key establishment | X25519/ECDH + PQ 계획 | textbook RSA·자체 DH |
| Signature | Ed25519/RSA-PSS + PQ 계획 | raw RSA·nonce 재사용 ECDSA |

---

# PART B. 해시에서 인증까지 (Cryptographic Hash)

> 핵심 질문: 무결성·인증·전자서명은 어디까지 같은가? → **"누가 계산할 수 있는가"**로 구분.
> 비밀 입력 기준: Hash(없음) / MAC·HMAC(공유키) / KDF(secret) / Signature(개인키) / KEM(개인·공개키)

## 1. 암호학적 해시 ⭐
- `h = H(m)`: 임의 길이 → **고정 길이 digest**. 키 없이 누구나 계산. 출력 고정이라 **충돌은 반드시 존재**.
- SHA-256 = 256-bit digest.
- 📝 필기(정확): **one-wayness** — `m → H(m)`은 쉬움(○), `H(m) → m`은 어려움(✗).

### 3가지 저항성 ⭐⭐ (second-preimage vs collision 구분 중요)
| 속성 | 공격자에게 고정된 값 | 찾아야 하는 값 |
|------|---------------------|----------------|
| **Preimage resistance** (역상) | digest h | m: H(m)=h |
| **Second-preimage resistance** (제2역상) | **원본 메시지 m** | m′≠m, H(m′)=H(m) |
| **Collision resistance** (충돌) | **고정 메시지 없음** | (m, m′): m≠m′, H(m)=H(m′) |
> 차이 핵심: second-preimage는 **특정 원본 m이 고정**, collision은 **아무 두 개나** 같으면 됨.

### 생일 역설(Birthday)과 충돌 강도 ⭐⭐ (수식 이해)
- n-bit 해시 → 출력 수 N=2ⁿ. q개 해시 중 충돌 쌍 ≈ q(q-1)/2 ≈ q²/2N.
- 50% 충돌 경계: `q ≈ √(2N ln2) = 1.177·2^(n/2)` → **n-bit 해시의 충돌 강도는 약 n/2 bits**.
- 고전 생일 역설: 23명이면 같은 생일 쌍 확률 ≈ 50.7% (쌍이 23×22/2=253개). "나와 같은"이 아니라 "아무 두 사람".
- 유도: `P(no collision) = Π(1-i/N) ≈ exp(-q²/2N)`, 충돌=1-그것.
- toy: 16-bit 해시 → 50% 경계는 약 1.177×2⁸ ≈ 301개 (충돌 강도 ≈ 8 bits).
> ⭐ **avalanche(1비트 입력 변화가 출력 절반을 바꿈)만으로 안전성 증명 못 함** — preimage/collision 공격, 구조적 약점, domain separation, 구현 맥락까지 평가해야.

### 해시 알고리즘 현황
| 알고리즘 | 상태 |
|----------|------|
| **MD5** | 실용적 collision 생성 → **사용 금지** |
| **SHA-1** (160-bit) | collision resistance 붕괴 → SHA-2/3로 전환 |
| **SHA-256** (SHA-2) | 현행 기본 |
| **SHA-3** (SHA3-256) | Keccak sponge 기반, 현행 대안·새 구조 |
- 📝 필기: SHA-1=160-bit, SHA-512도 존재. SHA-2 계열(224/256/384/512), SHA-3는 SHAKE 가변 길이 출력도 제공.

### 해시의 안전한/위험한 용도 ⭐
| 용도 | 일반 hash만으로? | 대안 |
|------|------------------|------|
| 다운로드 오류 확인 | 조건부 ○ (신뢰 채널로 digest 받으면) | — |
| **공격자 상대 메시지 인증** | ❌ | **HMAC** 또는 signature |
| **비밀번호 저장** | ❌ | **salt + memory-hard KDF** |
| 파일 식별·중복제거 | 조건부 ○ | 강한 hash |
> 📝 필기(정확): 로그인은 `B(server) → DB(ID, H(pwd))` 저장, 클라이언트 `H(pwd') + salt`. 해시만으론 부족 → salt+KDF.

## 2. MAC과 HMAC ⭐⭐

### 잘못 만든 MAC — Length Extension 공격 ⭐
- `tag = SHA-256(secret ‖ message)` (secret-prefix) 구조는 일부 해시에서 **안전하지 않음**.
- **원리**: 공격자가 secret을 몰라도, 알려진 digest를 **내부 상태처럼** 사용해 **뒤에 데이터(suffix)를 이어 붙인 새 tag**를 계산 (`secret ‖ message ‖ 원래 padding ‖ suffix`).
- 예: "받는사람=Bob,금액=1000" 뒤에 "금액=9000000"을 붙여 위조 (뒤 중복 field 우선 parser 가정).
- ⭐ **방어**: secret-prefix 직접 만들지 말고 표준 **HMAC** 또는 **AEAD** 사용.

### MAC
- `tag = MAC(K, m)`. 수신자가 같은 K로 검증. **기밀성 미제공**. 공유자끼리 **누가 썼는지 구분 못 함**(= shared-key authenticity). → 무결성 + 공유키 인증.

### HMAC ⭐ (구조와 수식)
- `HMAC(K, m) = H((K' ⊕ opad) ‖ H((K' ⊕ ipad) ‖ m))`
- **2단계**: inner hash `H((K'⊕ipad)‖m)` → outer hash `H((K'⊕opad)‖inner)`.
- K' = K를 블록 크기 B에 맞춘 키. RFC 2104: ipad=0x36×B, opad=0x5C×B (공개 고정값). SHA-256 B=64바이트.
- ⭐ **HMAC은 inner/outer hash를 분리하므로 length extension으로 연장 불가** (secret-prefix의 결함을 구조적으로 해결).
- 예: 이체 요청 amount 변경 → 공격자는 새 tag 계산할 K가 없어 서버가 거부. (단, HMAC은 내용을 숨기지 않음 → 기밀성 필요하면 AEAD)

## 3. KDF (Key Derivation Function) ⭐

- **정의**: secret·shared secret·password를 받아 **용도·길이에 맞는 key material** 도출.
- **왜?**: 하나의 값을 AES key·MAC key·송신/수신 key로 **그대로 재사용하지 않고** context·용도별로 분리(key separation).
- ⭐ **KDF는 없는 entropy를 새로 만들지 않는다** — 불균일성 정리, 공격 비용 상승, key separation 제공.

### 2계열 KDF ⭐ (혼용 금지)
| 구분 | 입력 | 목표 | 대표 |
|------|------|------|------|
| **Cryptographic KDF** | shared secret·key material (높은 entropy) | extract·expand·context별 분리 | **HKDF** |
| **Password KDF** | 사람이 정한 **낮은 entropy** password | 각 추측에 time·memory 비용 부과 | **Argon2id**, scrypt, PBKDF2 |
> ⭐ HKDF를 password hashing에 쓰거나, Argon2id를 세션 key schedule에 쓰지 않는다. (📝 필기: Cryptographic KDF는 "seed 생성에 사용")

### HKDF (Extract-and-Expand)
- `IKM(입력 secret + salt) → Extract → PRK → + info(context) → Expand → OKM`
- info에 protocol·role·purpose를 넣으면 **같은 IKM에서 다른 용도 key 분리**. salt는 비밀 아니어도 됨.
- 실제: TLS 1.3이 (EC)DHE shared secret + handshake context로 traffic key 유도. OpenSSH는 bcrypt_pbkdf로 passphrase→키.

## 4. 비밀번호 저장 (Password Hashing) ⭐⭐

- **문제**: 비밀번호는 **낮은 entropy**. DB 유출 시 공격자가 **offline으로 후보 반복 계산**. 빠른 해시(SHA-256)일수록 공격도 빨라짐.
- **방어 목표**: ① **salt**로 사전계산 공유 차단 ② **memory·time cost**로 각 추측 비용 상승.

### 4요소 (서로 대체 불가) ⭐
| 요소 | 비밀? | 저장 위치 | 목적 |
|------|-------|-----------|------|
| **Salt** | 아니오 | hash 레코드와 함께 | **사용자별 사전계산 분리** (필수) |
| **Pepper** | **예** | DB 밖 secret store/HSM | DB 단독 유출 시 공격 비용↑ (선택) |
| **Time cost** | 아니오 | parameter | 추측당 반복 연산↑ |
| **Memory cost** | 아니오 | parameter | GPU·ASIC 공격 비용↑ |
> ⭐ **salt는 필수, pepper는 선택**. salt는 비밀 아님(사용자마다 무작위 생성, 레코드와 저장). 같은 비밀번호도 salt가 다르면 다른 verifier.
> salt = 사전계산 분리 / cost = 추측 1회를 느리게. **역할이 다름**.

### Argon2 변형 ⭐
| 변형 | 메모리 참조 | 특징 |
|------|------------|------|
| Argon2i | password 무관(독립) | cache-timing 사이드채널 강함 |
| Argon2d | 값 의존(data-dependent) | GPU 병렬 강함, 사이드채널 민감 |
| **Argon2id** | 하이브리드(앞 절반 i, 나머지 d) | **사이드채널 저항 + brute-force 비용 둘 다. RFC 9106 권장** |
- parameter: m(memory), t(iterations), p(parallelism) — 서버에서 benchmark로 결정.
- 📝 필기(정확): "병렬 시간·메모리 없이 쓰게 함, Guess attack 불가능하게." 저장 레코드 `$argon2id$v=19$m=...,t=...,p=...$salt$verifier`는 **복호용 암호문이 아니라 검증값**.

### 로그인 검증 (복호가 아니라 '일치 확인') ⭐
- TLS가 record 복호 → 서버가 password 받음 → salt·cost·verifier 조회 → 입력으로 candidate verifier 계산 → **constant-time 비교** → session 발급.
- ⭐ "복호하지 않는다"의 대상은 **password verifier**. 파일 복호가 필요하면 password와 **분리된 data-encryption key** 사용.

### Password KDF 비교
| | Argon2id | scrypt | bcrypt | PBKDF2 |
|---|----------|--------|--------|--------|
| 등장 | 2015 | 2009 | 1999 | 2000 |
| Memory-hard | 예 | 예 | 제한적 | **아니오** |
| GPU/ASIC 저항 | **매우 높음** | 높음 | 보통 | 낮음 |
| 신규 적합성 | **가장 권장** | 좋은 대안 | 기존 유지 | 규제·호환 중요 시 |
- **PBKDF2**: HMAC을 c번 반복, 필요 길이만큼 블록(T₁∥T₂…) 이어붙임. memory-hard 아님.

## 5. OTP (일회용 비밀번호) ⭐
> ⚠️ **One-Time Password(OTP)는 2주차 One-Time Pad(OTP)와 약어만 같고 완전히 다른 개념!**

| 방식 | 동작 |
|------|------|
| **TOTP** (시간 동기) | 공유 secret + 현재 time step → 짧은 code. `T=floor((time-T₀)/30)`, `HOTP(K,T)` → 6자리 |
| **Challenge-response** | 서버가 매번 새 challenge R → client가 `HMAC(K,R)` 응답 (K는 안 보냄). R 재사용 금지 |
| **S/KEY hash chain** | 해시 one-way로 역순 공개 |

- **TOTP = HOTP의 counter에 시간 구간 번호를 넣은 것**. 장점: 별도 challenge 왕복 없음. 주의: clock drift·짧은 code brute force·phishing relay.
- **S/KEY**: client가 x에서 N번 해시, 서버는 Hᴺ(x)만 저장. 로그인마다 Hᴺ⁻¹(x) 전송 → 서버가 한 번 해시해 비교 → 저장값 교체. 관측자가 Hᴺ⁻¹(x)를 알아도 **one-way라 Hᴺ⁻²(x) 역산 불가**.

### 소프트웨어 무결성 실습 (해시 검증)
- `sha256sum --check` 또는 PowerShell `Get-FileHash 파일 -Algorithm sha256`로 공식 게시 SHA-256과 비교.
- ⭐ 단, **checksum도 함께 변조될 수 있으므로 전자서명이 더 강한 검증**.

---

## ✅ 3주차 primitive 요약 (입력·출력·목적)
| Primitive | 비밀 입력 | 출력 | 목적 |
|-----------|-----------|------|------|
| Hash | 없음 | digest | 지문·충돌/역상 저항 |
| HMAC | 공유 key | tag | 메시지 인증 |
| HKDF | shared secret | key material | context별 key 분리 |
| Argon2id | password | verifier/key | offline 추측 비용↑ |
| Signature | 개인키 | signature | 공개 검증 가능 인증 |
| KEM | 개인/공개키 | ct + secret | 공유 secret 설정 |
> **3주차 결론**: 암호 primitive는 계산식보다 **신뢰 관계**(누가 계산/검증/신뢰)를 정의하는 도구.
