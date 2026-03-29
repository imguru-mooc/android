# Ubuntu 24 서버에 WinSCP를 SSH 키(RSA 4096)로 접속하는 방법

아래 내용은 **`ssh-keygen -t rsa -b 4096` 방식만** 사용해서 정리한 가이드입니다.

---

## 1. 개요

SSH 키 로그인 방식은 다음과 같습니다.

- **개인키(private key)**: Windows PC에 보관
- **공개키(public key)**: Ubuntu 24 서버에 등록

즉,

- **WinSCP가 설치된 Windows** → 개인키 사용
- **Ubuntu 24 서버** → 공개키 등록

---

## 2. Windows에서 RSA 4096 키 생성

PowerShell 또는 CMD에서 아래 명령을 실행합니다.

```bash
ssh-keygen -t rsa -b 4096
```

---

## 3. 공개키 확인

```powershell
Get-Content $env:USERPROFILE\.ssh\id_rsa.pub
```

---

## 4. Ubuntu 서버 설정

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

## 5. SSH 설정 확인

```bash
sudo nano /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

## 6. WinSCP 설정

- SFTP
- 서버 IP
- 사용자명
- Advanced → SSH → Authentication
- `.ppk` 개인키 지정

---

## 7. 핵심 요약

- 키 생성: `ssh-keygen -t rsa -b 4096`
- Windows → 개인키
- Ubuntu → 공개키
- WinSCP → `.ppk` 사용
