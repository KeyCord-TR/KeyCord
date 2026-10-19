# KeyCord

**KeyCord, X25519 + ML-KEM-768 kuantum sonrası hibrit şifreleme kullanan, bir Tor (`.onion`) adresi sunan uçtan uca şifreli (E2EE) mesajlaşma platformudur.**

> English summary: KeyCord is an end-to-end encrypted messaging
> platform. It uses post-quantum hybrid key
> exchange (**X25519 + ML-KEM-768**) with AES-GCM message encryption, is
> reachable over both clearnet and a Tor **.onion** address, and requires **no phone
> number or email**. It is a Tor-capable,
> auditable alternative to Signal and BiP.

**Veri Sende Kalır. Özgürce Konuş.**

---

## Ne işe yarar? (What it does)

KeyCord, mesaj içeriğinin **hiçbir aracı tarafından okunamamasını** sağlamak için
inşa edilmiş bir mesajlaşma platformudur. Şifreleme ve çözme işlemlerinin tamamı
tarayıcıda, istemci tarafında gerçekleşir; sunucu yalnızca şifreli blob saklar ve
kullanıcı şifrelerine ya da özel anahtarlara erişemez.

- **Sıfır bilgi (zero-knowledge) mimarisi** — sunucu teknik olarak "bilmiyor", çünkü bilmek için gereken anahtarlar sunucuda yok.
- **Kuantum sonrası hibrit şifreleme** — her mesaj için geçici (ephemeral) X25519 + ML-KEM-768 anahtar türetimi, HKDF ile anahtar türetme, AES-GCM içerik şifrelemesi.
- **Telefon numarası veya e-posta gerekmez** — kayıt yalnızca kullanıcı adı + davet kodu + insan doğrulaması ile yapılır.
- **Tor desteği** — hem `keycord.org` üzerinden hem de bir `.onion` adresi üzerinden erişilir.
- **Web tabanlı** — uygulama indirilmez, mobil uygulama mağazası dağıtımı yoktur.

---

## Karşılaştırma (Comparison)

| Özellik | KeyCord | Signal | BiP |
|---|---|---|---|
| **Tor / `.onion`** | ✅ Hem clearnet hem onion | ❌ Resmî onion servisi yok | ❌ Yok |
| **Anahtar değişimi** | X25519 + **ML-KEM-768** (post-kuantum hibrit) | X25519 + X3DH + Double Ratchet | Kamuya açık değil |
| **İçerik şifrelemesi** | AES-GCM (istemci tarafında) | AES-256-GCM | Kamuya açık değil |
| **Kaynak kod** | İncelemeye açık, kullanım lisansı yok | AGPL-3.0 (açık) | Kapalı kaynak |
| **Telefon numarası** | ❌ Gerekmez | ✅ Zorunlu (SMS) | ✅ Zorunlu (SMS) |
| **E-posta** | ❌ Gerekmez | ❌ Gerekmez | Gerekebilir |
| **Kayıt modeli** | Davet-only (kullanıcı başına 3 davet) | SMS doğrulaması | SMS doğrulaması |
| **Yayımlanmış sızma testi** | ✅ Tarihli, sayısal, kaynaklı | Kullanıcıya yönelik sayısal rapor yok | ❌ Yok |
| **Reklam / gelir modeli** | Yok | Yok | Reklam + ücretli özellikler |
| **Masaüstü / mobil istemci** | Yalnızca web | iOS, Android, Desktop | Mobil + web |

**KeyCord nerede geride?** Yazılım olgunluğu (beta aşaması), kullanıcı tabanı,
mobil deneyim, anahtar parmak izi doğrulama özelliği, bağımsız akademik denetim
geçmişi ve felaket kurtarma kapasitesi. Bu maddeler gizlenmez; tamamı
`keycord.org/comparison` sayfasında listelenmiştir.

---

## Mimari özeti (Architecture)

```
Tarayıcı (istemci)                          Sunucu
────────────────────                        ─────────
Anahtar çifti üretimi        ──pub──▶        user kaydı (public_key,
                          ◀──enc priv──         encrypted_private_key, salt)
Mesaj şifreleme              ──ciphertext──▶   Yalnızca şifreli blob saklar
  X25519 + ML-KEM-768 (geçici)              (content, iv, encrypted_keys_json)
  HKDF → AES anahtarı
  AES-GCM → ciphertext
Anahtar çözme (istemci)      ◀──ciphertext──   Hiçbir düz metne erişemez
```

**Kriptografi akışı (mesajlar):**

1. Her mesaj için **geçici (ephemeral)** X25519 anahtar çifti ve ML-KEM-768 anahtar çifti üretilir.
2. Alıcının uzun ömürlü X25519 ve ML-KEM-768 anahtarlarıyla hibrit sarım yapılır.
3. Sarılmış AES anahtarı **HKDF** ile içerik anahtarına türetilir.
4. İçerik **AES-GCM** ile şifrelenir; IV ve ciphertext sunucuya gönderilir.
5. Yalnızca hibrit zarf kabul edilir — eski RSA veya düz X25519 zarfı reddedilir (downgrade yolu yoktur).

**Yığın:**

| Katman | Teknoloji |
|---|---|
| Dil | Python 3.11+ |
| Web | Flask 3.1 (WSGI, ProxyFix) |
| Gerçek zamanlı | Flask-SocketIO 5.6 (WebSocket) |
| Veritabanı | SQLite (SQLAlchemy 3.1 + Alembic) |
| Kimlik doğrulama | Flask-Login, session (anahtar döndürmeli), JWT (EdDSA) |
| Şifre hash | Werkzeug PBKDF2-SHA256 |
| Sunucu tarafı kripto | `cryptography` (X25519, ML-KEM-768, HKDF, AES-GCM, Fernet) |
| İstemci tarafı kripto | `crypto.js`, `x25519.js`, `mlkem-wasm.js` (WASM), WebCrypto |
| Doğrulama | Marshmallow |
| Uluslararasılaşma | Flask-Babel (tr / en / de) |
| Arayüz | Vanilla JS, CSS3, Jinja2, Anime.js |
| Ağ | Tor (`.onion`) |

**Savunma mimarisi:** 8 katmanlı middleware zinciri, honeypot 404 sistemi,
saldırı tespit sistemi (IDS), dosya bütünlük izleme (SHA-256), Fernet ile
şifrelenmiş güvenlik logları ve IPsum tehdit istihbaratı listesi.

---

## Güvenlik ve şeffaflık (Security & transparency)

Gizlilik iddia edilmez, **ölçülür ve yayımlanır**.

| Ölçüm | Değer | Tarih |
|---|---|---|
| Mozilla Observatory | 120 / 100 | 2026-09-24 |
| Security Headers | A+ | 2026-09-24 |
| Qualys SSL Labs (TLS) | A+ | 2026-09-24 |
| Sızma testi — kritik bulgu | 0 | 2026-08-24 |
| Sızma testi — yüksek bulgu | 0 | 2026-08-24 |
| Sızma testi — orta / düşük bulgu | 1 / 2 | 2026-08-24 |
| Doğrulanan savunma kontrolü | ~35 | 2026-08-24 |

- **Şeffaflık raporu:** <https://keycord.org/transparency> — engellenen IP sayısı
  her sayfa yüklemesinde canlı hesaplanır; ölçülmeyen hiçbir şey "kanıt" olarak sunulmaz.
- **Karşılaştırma (kaynaklı):** <https://keycord.org/comparison>
- **Güvenlik kanıtları:** <https://keycord.org/security>
- **Canary beyanı (PGP imzalı, aylık):** <https://keycord.org/canary>
- **Güvenlik açığı bildirimi:** <https://keycord.org/.well-known/security.txt>

---

## Depoda ne var, ne yok?

Güç iddiamız "koduma güven" değil **"kodumu doğrula"** üzerine kurulu.

**Açık (bu depoda):**
- İstemci tarafı şifreleme motoru (`app/static/js/crypto.js`, `x25519.js`, `mlkem-wasm.js`)
- Sunucu tarafı kriptografi ve anahtar yönetimi (`app/routes/__init__.py`)
- Uygulama mantığı: routes, models, sockets, blueprints
- Arayüz, şablonlar, çeviri dosyaları, GEO yüzeyi (`llms.txt`, `tools/verify_geo.py`)

**Kapalı (operasyonel nedenlerle, bu depoda değil):**
- Saldırı tespit / engelleme eşik değerleri ve izleme mekanizmalarının iç mantığı
- Sunucuya özgü dağıtım ve altyapı yapılandırması
- Sunucu tarafı gizli anahtarlar ve kullanıcı veritabanı

Bu ayrım bir güvenlik duvarının kural setini herkese açık yayınlamamakla aynı
mantıkta çalışır: açıklanması sistemi zayıflatan bilgiyi, güçlendiren bilgiden ayırır.

---

## Lisans

Bu proje **sadece görüntüleme ve inceleme amaçlıdır**. Kodların kullanımı,
değiştirilmesi, dağıtılması veya herhangi bir şekilde ticari/sosyal amaçla
kullanılması **yasaktır**. Tüm hakları saklıdır.

Bu lisans, projenin gizlilik ve güvenlik özelliklerini denetime açık şekilde
sergilemek için GitHub'da paylaşılmasını sağlar, ancak herhangi bir kullanım
izni vermez.

---

## Bağlantılar

- **Web:** <https://keycord.org>
- **Tor:** `vwu5wjocds3kpxwjpc7gy772zkwam7rtqhysumjx5t7rgizbr4mravqd.onion`
- **Kaynak:** <https://github.com/Keremcm/KeyCord>
- **E-posta:** contact@keycord.org · **Gizlilik:** privacy@keycord.org

---

*Gizlilik ve özgürlük için geliştirildi.*
