# CC — Cross-Chain Stablecoin Settlement

크로스체인 기반 브릿지 · 스왑 · 온오프램프 PoC

---

## 개요

Ethereum Sepolia와 GIWA Chain 간 USDC ↔ wUSDC ↔ KRWS 크로스체인 정산 시스템입니다.  
MetaMask 서명 기반 인증, 온체인 Oracle 환율, 은행 OpenAPI 연동을 통해  
브릿지 · 환전 · 원화 입출금을 단일 플로우로 구현합니다.

---

## 핵심 기술

| 영역 | 구현 내용 |
|---|---|
| **Bridge** | Sepolia USDC Lock → GIWA wUSDC Mint / Burn → Sepolia USDC Unlock |
| **Swap (Oracle)** | FxRateOracle 온체인 환율 적재 → StableSwapRouter burn+mint 단일 TX |
| **On-Ramp** | 은행 OpenAPI 원화 출금 → GIWA KRWS Mint |
| **Off-Ramp** | GIWA KRWS Burn → 은행 OpenAPI 원화 입금 |
| **Rollback** | 단계별 실패 시 즉시 원상복구 (@Retryable + @Recover) |

---

## 시스템 구성

```
[Sepolia Testnet]       [GIWA Chain ID:91342]     [Banking]
USDCBridgeVault         wUSDC / KRWS              OpenAPI
      │                 StableSwapRouter              │
      │   cc-relayer    FxRateOracle                  │
      └─────────────────────────────────────────      │
                   cc-fx │ cc-ramp ────────────────────
                         │
                  [PostgreSQL]
                         │
           cc-admin ─────┴───── cc-app
           cc-admin-web         (고객 웹)
```

---

## 서비스 구성

| 서비스 | 포트 | 역할 |
|---|---|---|
| `cc-relayer` | 8081 | 회원가입 · Bridge In/Out 릴레이어 |
| `cc-ramp` | 8082 | On/Off-Ramp |
| `cc-fx` | 8083 | 환율 Oracle · Swap |
| `cc-admin` | 8084 | 운영 어드민 API |
| `cc-app` | 3000 | 고객 React 앱 |
| `cc-admin-web` | 3001 | 어드민 React 앱 |

---

## 기술 스택

**Backend** Java 21 · Spring Boot 3.2 · Spring Data JPA · Spring Retry  
**Blockchain** Web3j · Solidity 0.8 · Hardhat · MetaMask personal_sign  
**Frontend** React 18 · Vite · ethers.js  
**DB** PostgreSQL 15  
**Infra** Docker · GitHub Actions · Oracle Cloud

---

## 트랜잭션 흐름

```
Bridge In
  고객 MetaMask → USDC approve + vault.lock()
  → Relayer HTTP polling 감지 → wUSDC mint (GIWA)
  → 실패 시: vault.rollbackUnlock() → USDC 반환

Swap
  wUSDC burn + KRWS mint 단일 TX (EVM 자동 원자적 롤백)
  FxRateOracle 온체인 환율 기준 실행

On-Ramp
  은행 OpenAPI 원화 출금 → KRWS mint (GIWA)
  → 실패 시: 뱅킹 역방향 환불

Off-Ramp
  KRWS burn (GIWA) → 은행 OpenAPI 원화 입금
  → 실패 시: KRWS mint 복원

Bridge Out
  wUSDC burn (GIWA) → vault.unlock() → USDC → MetaMask
  → 실패 시: wUSDC mint 복원
```

---

## 컨트랙트

| 컨트랙트 | 체인 | 역할 |
|---|---|---|
| `USDCBridgeVault` | Sepolia | USDC Lock / Unlock 에스크로 |
| `wUSDC` | GIWA | Wrapped USDC (6자리) |
| `KRWS` | GIWA | KRW 스테이블코인 (2자리) |
| `StableSwapRouter` | GIWA | wUSDC ↔ KRWS 환전 |
| `FxRateOracle` | GIWA | 환율 온체인 저장 |

---

## GitHub Actions CI/CD

`main` 브랜치 푸시 시 자동 빌드 → Docker 이미지 → Oracle Cloud 배포
*마이너한 개발시엔 신규 브랜치 혹은 로컬에서 개발 권장

```bash
git push origin main
```