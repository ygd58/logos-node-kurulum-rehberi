# Logos Testnet Node Kurulum Rehberi (Türkçe)

Logos testnet node'unu komut satırından kurmak ve çalıştırmak için adım adım Türkçe rehber.

> **Not:** Bu, resmî olmayan bir topluluk rehberidir. Adımlar [resmî dokümana](https://docs.logos.co/blockchain/get-started/run-blockchain-node-from-cli) (Testnet v0.3) dayanır. Sürüm numaraları ve bootstrap peer adresleri değişebilir, kurmadan önce resmî dokümandaki güncel değerleri kontrol et.
>
> Son güncelleme: 2026-11-27

## Başlamadan önce

Node için 2 çekirdekli, 1 GB RAM'li, SSD'de 1-2 GB boş yeri olan bir Linux sunucu yeter. Asıl dikkat edilecek iki şey CPU'nun ADX desteği ve glibc sürümü.

| Gereksinim | Değer |
| --- | --- |
| CPU | 2 çekirdek, 2 GHz veya üstü; x86_64'te ADX desteği şart |
| RAM | En az 1 GB |
| Disk | 1-2 GB boş, SSD |
| İşletim sistemi | Linux x86_64, macOS aarch64 veya Raspberry Pi 5 |
| glibc (Linux) | 2.39 veya üstü (resmî şart, aşağıdaki nota bak) |
| Ağ | Yaklaşık 1 Mbps boş bant genişliği |

**1. ADX kontrolü.** Komut `adx` yazdırmazsa CPU uygun değildir ve node `signal 4` ile çöker:

```bash
grep -o adx /proc/cpuinfo | head -1
```

Sanal sunucu kullanıyorsan sağlayıcıdan host CPU özelliklerinin geçirilmesini iste. `kvm64` ve `qemu64` gibi genel modeller ADX'i gizler.

**2. glibc kontrolü.** Çıkan sürüm 2.39 veya üstü olmalı (Ubuntu 24.04 ve üzeri bu şartı karşılar):

```bash
ldd --version | head -1
```

> Not: Biz `blockchain_module 0.3.2` sürümünü Ubuntu 22.04'te çalıştırdık. 2.39 şartı resmî dokümandaki (0.3.4) değerdir, eski sürümlerde çalışabilir ama garanti değil.

**3. FUSE paketleri.** `logosctl` bir AppImage olduğu için gerekir:

```bash
sudo apt update
sudo apt install -y fuse libfuse2t64 curl jq
```

`jq`, ilerideki kontrol komutlarının çıktısını okunur yapmak için kullanılır.

## Adım 1: logosctl ve blockchain modülü

Önce `logosctl` aracını kur, sonra daemon'u başlatıp blockchain modülünü yükle. Her şey normal (root olmayan) bir kullanıcıyla yapılır; node dosyaları o kullanıcının ev dizininde oluşur.

**1. logosctl kurulumu:**

```bash
curl -fsSL https://raw.githubusercontent.com/logos-co/logos-docs/main/resources/scripts/install-logosctl.sh | sudo sh
```

**2. Daemon'u başlat.** Node dosyalarının duracağı dizinden başlat (ev dizini uygun):

```bash
cd ~
logosctl daemon start
```

**3. Yeni bir terminal aç** (aynı kullanıcıyla) ve katalogları yenile. Daemon önceden çalışıyor olsa bile bunu yap:

```bash
logosctl catalog refresh
```

**4. Modül paketini kur.** Resmî dokümanın yazdığı sürüm 0.3.4'tür; daha yeni bir sürüm çıkmış olabilir, kurmadan önce [resmî dokümandaki](https://docs.logos.co/blockchain/get-started/run-blockchain-node-from-cli) güncel sürüm numarasına bak ve `--version` değerini ona göre ayarla:

```bash
logosctl package install blockchain_module \
  --version 0.3.4 \
  --yes
```

0.3.2'den güncelliyorsan mevcut `user_config.yaml` dosyanı sakla. İlk başlatışta yeni sürüm onu dönüştürür ve eskisini `user_config.yaml.0.3.2.bak` olarak yedekler.

**5. Modülü yükle:**

```bash
logosctl module load blockchain_module
```

Daemon hazır olmadan çalıştırırsan RPC veya eksik istemci yapılandırması hatası alırsın. `logosctl status` ile kontrol edip tekrar dene.

## Adım 2: Yapılandırma ve başlatma

Bu adım iki dosya üretir: `user_config.yaml` (ayarlar) ve `keystore.yaml` (gizli anahtarlar). Dosyalar daemon'un çalışma dizinine yazılır; rehberdeki gibi başlattıysan ev dizinin (`$HOME`).

**1. Yapılandırmayı üret.** Aşağıdaki bootstrap peer listesi 0.3.4 sürümüne aittir. Peer adresleri sürümle değişebilir, bu yüzden kurmadan önce güncel sürüm notlarındaki listeyi kullan:

```bash
logosctl call blockchain_module generate_user_config '{
    "initial_peers": [
        "/ip4/65.109.51.37/udp/3000/quic-v1/p2p/12D3KooWFrouXfmrR4nsLMtE7wu15DoMJ6VtoUtHinREZCvbWHar",
        "/ip4/65.109.51.37/udp/3001/quic-v1/p2p/12D3KooWJRGau8M1rjT7R5e4YYsgdFhsMX35nRDtMwCDjxQkXAHz",
        "/ip4/65.109.51.37/udp/3002/quic-v1/p2p/12D3KooWQXJavMDTRscjauFSgVAB1VLB6Rzpy2uY5SU9Tk7927tb",
        "/ip4/65.109.51.37/udp/50001/quic-v1/p2p/12D3KooWSQc7CcGtvWDPF1yCbBthFnQjprfCVHmfmNDUrSmqQsU1"
    ]
}'

chmod 600 user_config.yaml keystore.yaml
```

> **`keystore.yaml` gizli anahtarlarını içerir.** Kimseyle paylaşma, ekran görüntüsüne ya da sohbete koyma, yedeğini güvenli bir yerde sakla. Her node için yeni bir çift üret, başka bir node'un dosyasını kopyalama.

Üretilen yapılandırma yalnızca varsayılandan farklı değerleri (verdiğin peer'leri) listeler. Başka bir ayarı değiştirmek için tam yoluyla ekle; örneğin P2P portunu değiştirmek için:

```yaml
network:
    backend:
    swarm:
        port: 3000
```

**2. Node'u başlat:**

```bash
logosctl call blockchain_module start "$HOME/user_config.yaml" ""
```

Sondaki boş `""` argüman bilinçli; artık ayrı bir `deployment.yaml` gerekmez. Daemon'u ev dizininden başlatmadıysan yolu mutlaka tam (mutlak) yaz.

**Bilmen gereken başlıca ayarlar:**

| Ayar | Varsayılan | Not |
| --- | --- | --- |
| `network.backend.swarm.port` | `3000` (UDP) | Herkese açık P2P portu; güvenlik duvarıyla uyumlu olmalı |
| `api.backend.listen_address` | `127.0.0.1:8080` | Yerel API, dışarıya açma |
| `state.base_folder` | `./state` | Kalıcı bir yerel yol kullan |
| `tracing.level` | `INFO` | Gözetimsiz çalışma için uygun |

Loglar saatlik dosyalara yazılır (`logos-blockchain.log.<tarih-saat>`), güncel dosyaya `logos-blockchain.log.latest` bağlantısı işaret eder.

## Adım 3: Node'un çalıştığını doğrula

Node ilk başladığında zinciri indirir (`Bootstrapping`) ve yaklaşık 1 saat sonra `Online` olur. Bu sürede yeniden başlatma, eşitleme ilerlemesini sıfırlar.

**1. Konsensüs durumu:**

```bash
logosctl call blockchain_module get_cryptarchia_info | jq -r .result.value | jq .
```

Yerel HTTP uç noktasıyla da bakabilirsin:

```bash
curl -s http://localhost:8080/cryptarchia/info | jq .
```

`mode` (CLI) veya `state` (HTTP) önce `Bootstrapping` gösterir, eşitlenince `Online` olur. `slot` ve `height` değerlerinin arttığını doğrula; yaklaşık her 10 saniyede yeni blok beklenir.

**2. Eşler (peers):**

```bash
curl -s http://localhost:8080/network/info | jq .
```

`n_peers` 0'dan büyük olmalı.

**3. 30-60 saniye sonra** `get_cryptarchia_info` komutunu tekrar çalıştır ve `slot` ile `height` değerlerinin arttığını gör.

> **Dikkat:** Node `Online` olmadan `pow_status` veya `pow_claimable_rewards` çağırma. Bu çağrılar eşitleme sırasında asılı kalır ve sonraki `logosctl` komutları `logosctl daemon stop` ile daemon'u durdurup yeniden başlatana kadar hata verir. (Bu davranış, [logos-blockchain #3780](https://github.com/logos-blockchain/logos-blockchain/issues/3780) numaralı konuda da bildirilmiş.)

## Adım 4-5: Madencilik ve lider ödülleri

Eşitlenmiş bir node zinciri doğrular ama bakiyesi yoksa blok önermez. Bakiyeyi madencilikle (PoW) kazanırsın; ödüller `keystore.yaml` içindeki `public_keys.PoWClaim` anahtarına gider. Bu adımların hepsi node `Online` olduktan sonra yapılır.

**1. Madenciliği başlat.** Madencilik varsayılan olarak kapalıdır ve **her yeniden başlatmadan sonra yeniden başlatılması gerekir**. Ödül talebi (auto-claim) kendiliğinden başlar, ayrı komut gerekmez:

```bash
logosctl call blockchain_module pow_start_mining
```

**2. Durumu kontrol et:**

```bash
logosctl call blockchain_module pow_status | jq -r .result.value | jq .
```

**3. Bakiyeyi gör.** `<anahtar>` yerine önceki çıktıdaki `public_key` değerini yaz:

```bash
curl -s http://localhost:8080/wallet/<anahtar>/balance | jq .
```

**4. Yeterli bakiyeye ulaşınca madenciliği durdur** (isteğe bağlı):

```bash
logosctl call blockchain_module pow_stop_mining
```

Token'lar konsensüse katılmaya 3,5 saat sonra uygun olur. Blok önerme olasılıklıdır ve senin payının toplam aktif paya oranına bağlıdır.

### Lider ödüllerini talep et

Node'un önerdiği her blok bir lider ödülü fişi (voucher) üretir. CLI node'larında bunları elle talep etmen gerekir:

```bash
logosctl call blockchain_module wallet_get_claimable_vouchers | jq -r .result.value | jq .
logosctl call blockchain_module leader_claim
```

`leader_claim` her çağrıda bir voucher talep eder. Birden fazlası varsa her biri için ayrı çağır, aralara birkaç saniye koy. `"success": false` ve "No claimable voucher found" yanıtı şimdilik talep edilecek bir şey olmadığı anlamına gelir, sonra tekrar dene. Node `Online` olmadan ve stake olgunlaşmadan `leader_claim` çalıştırma.

## Portlar, güvenlik ve sık hatalar

Dışarıya yalnızca 3000/UDP açık olmalı; API portu (8080) asla açılmamalı.

| Port | Yön | Ne için |
| --- | --- | --- |
| 3000/UDP (QUIC) | Gelen, internete açık | P2P; node'un diğer eşlerle konuşması için |
| 8080 (HTTP) | Yalnızca yerel (`127.0.0.1`) | Node API'si; dışarıya açma |
| 3400/UDP | Gelen | Yalnızca Blend Ağı'na katılacaksan |
| Giden UDP | Giden | Bootstrap peer'lere ulaşmak için serbest olmalı |

UFW kullanıyorsan:

```bash
sudo ufw allow 3000/udp
```

**Güvenlik kuralları**

- `keystore.yaml` dosyasını asla paylaşma ve yedeğini sunucu dışında güvenli bir yerde tut.
- Node dinamik cüzdan anahtarı yönetimini desteklemez. Anahtar eklemek için `keystore.yaml` dosyasını elle düzenleyip node'u yeniden başlat.
- Modüller daemon yeniden başladıktan sonra kalıcı değildir; `module load` komutunu tekrar çalıştır.

**Sık görülen hatalar**

| Belirti | Ne yapılmalı |
| --- | --- |
| `RPC call failed` | `logosctl daemon status` ile bak, gerekirse `logosctl daemon start --detach` ile başlat, modülü yeniden yükle |
| `signal 4` ile çökme | CPU'da (veya sanal CPU modelinde) ADX yok; yukarıdaki kontrole bak |
| Uzun süre `Bootstrapping`, tekrarlayan `FutureBlock` hataları | Modülü kaldırıp yeniden yükle, sonra `start user_config.yaml ""` ile başlat |
| Başlangıçta `AllPeersFailed` | Giden UDP'yi ve güncel bootstrap peer listesini kontrol et, modülü yeniden başlat |

## Uzun süreli işletme: yeniden başlatma ve takılma sorunu

Bu bölüm resmî dokümanda yok; Logos 0.3.2 çalıştıran kendi node'umuzda gözlemlediklerimize dayanır ve diğer sürümlerde geçerli olup olmadığını doğrulamadık.

**Yeniden başlatma sonrası kontrol listesi.** Sunucu veya daemon yeniden başladıktan sonra node kendiliğinden geri gelmez; sırayla şunları yap:

1. `logosctl daemon start --detach` ile daemon'u başlat.
2. `logosctl module load blockchain_module` ile modülü yükle.
3. `logosctl call blockchain_module start "$HOME/user_config.yaml" ""` ile node'u başlat.
4. `get_cryptarchia_info` çıktısı `Online` olana kadar bekle.
5. `logosctl call blockchain_module pow_start_mining` ile madenciliği yeniden başlat.

**Ödül talebi takılması (claim stall).** Bizim node'da birkaç saatte bir şu tablo oluştu: loglarda `Claimed N PoW reward(s)` satırları gelmeye devam ediyor ama `Retired N settled PoW claim(s)` satırları duruyor ve bakiye donuyor. Modülü yeniden başlatmak (yukarıdaki liste) sorunu geçici olarak çözüyor. Bu davranışı ölçümlerle [logos-blockchain #4203](https://github.com/logos-blockchain/logos-blockchain/issues/4203) numaralı konuda bildirdik; kesin kök neden henüz bilinmiyor.

**Takıldığını nasıl anlarsın.** Bakiyeyi 30 dakika arayla iki kez oku; değişmiyorsa ve node `Online` ise şüphelen. Loglarda en son `Retired` satırının ne zaman geldiğine bakmak daha net sonuç verir:

```bash
grep -a "Retired" logos-blockchain.log.latest | tail -1
```

Özet olarak, en son `Retired` satırı 15-20 dakikadan eskiyse node'u yukarıdaki listeyle yeniden başlat. Bizde bunu her 5 dakikada bir kontrol edip gerekirse yeniden başlatan bir cron betikle (15 dakikalık eşik, en fazla saatte bir yeniden başlatma) otomatik hale getirdik. Betik kuruluma özel olduğu için buraya koymadık; ilgilenirsen bir issue aç, paylaşırız.

**Bir hatada yardım isterken** loglardaki `ERROR` ve `WARN` satırlarını, node sürümünü ve `logosctl --version` çıktısını ekle; `keystore.yaml` içeriğini asla paylaşma.

## Katkı

Hata, eksik ya da yanlış bir şey görürsen issue aç veya pull request gönder. Sürümler değiştikçe rehberi güncel tutmak için yardım çok değerli.

## Lisans

[MIT](LICENSE)
