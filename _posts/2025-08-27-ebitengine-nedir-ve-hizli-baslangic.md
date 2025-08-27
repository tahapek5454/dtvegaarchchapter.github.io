---
layout: post
title: "Ebitengine Nedir? Hızlı Başlangıç, Oyun Döngüsü ve Görsel Çizimi"
categories: [oyun, golang]
tags: [ebitengine, ebiten, golang, 2d, wasm, android, ios]
lang: tr
author: QuickOrBeDead
excerpt: Ebitengine (Ebiten) ile 2D oyun geliştirmeye giriş; nedir, nerede kullanılır, kurulum, game loop, görsel çizmek, desteklenen platformlar, yardımcı kütüphaneler ve kaynaklar.
date: 2025-08-27
last_modified_at: 2025-08-27 10:18:00 +0300
---

## TL;DR

- Ebitengine (eski adıyla Ebiten), Go ile 2D oyun geliştirmek için kullanılan hafif ve çoklu platform destekli bir oyun motorudur.
- Windows, macOS, Linux, WebAssembly (WASM), Android ve iOS desteği.
- Kurulum: `go get github.com/hajimehoshi/ebiten/v2`.
- Yardımcı kütüphaneler: `ebiten/inpututil`, `ebiten/text`, `ebiten/audio`, `ebitenui`, `donburi`, `mizu`.

---

## Ebitengine Nedir?

Ebitengine, Go dilinde 2D oyun geliştirmeyi hızlandıran, minimal ama işlevsel bir oyun motorudur. Öğrenmesi kolay bir API sunar, platform bağımsız çalışır ve varsayılan olarak sabit bir güncelleme hızında (60 TPS) oyun döngüsünü yönetir. Oyun geliştirme için gerekli temel taşları (rendering, input, ses, zamanlama) sağlar.

Not: Projenin adı son yıllarda Ebiten'den Ebitengine'e evrilse de Go modül yolu hâlâ `github.com/hajimehoshi/ebiten/v2` şeklindedir.

## Nerelerde Kullanılır?

- Küçük/orta ölçekli 2D oyunlar, Game jam'ler, prototipler
- Basit araçlar, simülasyonlar, görselleştirme uygulamaları
- Go ile hızlı deney yapmak isteyen ekipler ve bireysel geliştiriciler

## Kullanan Oyunlar ve Örnekler

Toplulukta pek çok game jam projesi ve bağımsız oyun Ebitengine ile geliştiriliyor. Başlamak için resmi örneklere atın:

- GitHub (örnekler klasörü): [hajimehoshi/ebiten/examples](https://github.com/hajimehoshi/ebiten/tree/main/examples)
- Resmi site ve dokümantasyon: [ebitengine.org](https://ebitengine.org/)

### Ebitengine ile geliştirilmiş bazı oyunlar

- Fishing Paradiso — [Odencat](https://odencat.com/fishingparadiso/en.html)
- Bear's Restaurant — [Odencat](https://odencat.com/bearsrestaurant/switch/en.html)
- From Madness with Love — [Playism sayfası](https://playism.com/en/game/frommadness-withlove/)
- Coral & The Abyss — [Intrugli Games](https://www.intrugligames.com/coral-and-the-abyss.html)
- Snowman Story — [Odencat](https://odencat.com/snowman/en.html)
- Inside the Crystal Mountain — [Intrugli Games](https://www.intrugligames.com/inside-the-crystal-mountain.html)
- Meg's Monster — [Odencat](https://odencat.com/bakemono/en.html)
- Rakuen — [Morizora Studios](https://www.laurashigihara.com/rakuen-en)
- Dr. Kobushi's Labyrinthine Laboratory — [Resmi site](https://drkobushi.com/)

### Ekibimizin Ebitengine ile Geliştirdiği Oyunlar

- Wordle — [2d-games/wordle/golang](https://github.com/DTVegaArchChapter/GameProgramming/tree/main/2d-games/wordle/golang)
- Blocks — [2d-games/blocks/golang](https://github.com/DTVegaArchChapter/GameProgramming/tree/main/2d-games/blocks/golang)
- Jigsaw Puzzle — [2d-games/jigsaw-puzzle/golang](https://github.com/DTVegaArchChapter/GameProgramming/tree/main/2d-games/jigsaw-puzzle/golang)

## Kurulum (Nasıl Yüklenir?)

Önkoşullar: Go (güncel sürüm).

Adımların özeti:

1) Yeni bir klasörde Go modülü başlatın.
2) Ebitengine paketini ekleyin.
3) Aşağıdaki "Minimal Oyun Döngüsü" örneğini `main.go` olarak kaydedip çalıştırın.

İsteğe bağlı komutlar:

```bash
# 1) Proje klasörü ve modül
mkdir hello-ebitengine; cd hello-ebitengine
go mod init example.com/hello-ebitengine

# 2) Ebitengine bağımlılığı
go get github.com/hajimehoshi/ebiten/v2

# 3) Çalıştırma
go run .
```

## Minimal Oyun Döngüsü (Game Loop)

Ebitengine oyun döngüsü 3 metottan oluşan bir arayüzle (Update, Draw, Layout) yönetilir. `Update` oyun durumunu ilerletir, `Draw` ekrana çizer, `Layout` hedef iç boyutu belirler.

```go
package main

import (
    "log"

    "github.com/hajimehoshi/ebiten/v2"
)

// ebiten.Game interface'ini implemente eder.
type Game struct{}

// İçerisinde Game state'ini güncelleyeceğimiz fonksiyon.
// Update fonksiyonu varsayılan olarak saniyede 60 defa çağırılır.
func (g *Game) Update() error {
    return nil
}

// Draw fonksiyonu içerisinde oyun nesneleri oyun ekranı üzerinde çizdirilir.
// 60 Hz'lik görüntülemede saniyede 60 kere çağırılır
func (g *Game) Draw(screen *ebiten.Image) {

}

// outsideWidth ve outsideHeight window'un boyutlarıdır.
// Sabit bir boyut dönebiliriz ya da outsideWidth ve outsideHeight üzerinden hesaplama yapıp değer dönebiliriz.
// screenWidth, screenHeight ile outsideWidth, outsideHeight farklı olsa bile görüntü window'a sığacak şekilde otomatik olarak ölçeklenir.
func (g *Game) Layout(outsideWidth, outsideHeight int) (screenWidth, screenHeight int) {
    return 320, 240
}

func main() {
    game := &Game{}
    // Ekran boyutu ayarlanır
    ebiten.SetWindowSize(640, 480)
    ebiten.SetWindowTitle("Your game's title")
    // Game loop'u başlatmak için ebiten.RunGame fonksiyonu çağırılır.
    if err := ebiten.RunGame(game); err != nil {
        log.Fatal(err)
    }
}
```

## Görsel Çizmek (Draw Image)

Aşağıdaki örnekte bir görseli belleğe gömüp (`go:embed`) decode ederek ekrana çiziyoruz. Geliştirme sırasında basitçe `ebitenutil.NewImageFromFile` ile dosyadan da yükleyebilirsiniz.

```go
package main

import (
    "embed"
    "bytes"
    "image"
    _ "image/png"
    "log"

    "github.com/hajimehoshi/ebiten/v2"
    "github.com/hajimehoshi/ebiten/v2/ebitenutil"
)

//go:embed assets/logo.png
var logoPNG []byte

type Game struct {
    logo *ebiten.Image
}

func (g *Game) Update() error { return nil }

func (g *Game) Draw(screen *ebiten.Image) {
    if g.logo == nil {
        // İlk çizimde embed edilmiş görseli decode edip Ebiten Image'a çeviriyoruz
        img, _, err := image.Decode(bytes.NewReader(logoPNG))
        if err != nil {
            log.Println("decode error:", err)
            return
        }
        g.logo = ebiten.NewImageFromImage(img)
    }

    if g.logo != nil {
        op := &ebiten.DrawImageOptions{}
        op.GeoM.Translate(100, 100) // ekranda konumlandırma
        screen.DrawImage(g.logo, op)
    }

    ebitenutil.DebugPrint(screen, "Logo (100,100) konumunda çizildi")
}

func (g *Game) Layout(outsideWidth, outsideHeight int) (int, int) { return 480, 320 }

func main() {
    game := &Game{}
    
    ebiten.SetWindowSize(960, 640)
    ebiten.SetWindowTitle("Draw Image Örneği")
    
    if err := ebiten.RunGame(game); err != nil {
        log.Fatal(err)
    }
}
```

Yerel dosyadan yüklemek için hızlı alternatif:

```go
img, _, err := ebitenutil.NewImageFromFile("assets/logo.png")
if err != nil { log.Fatal(err) }
g.logo = img
```

Notlar:

- `go:embed` ile derleme zamanında dosyaları binary içine gömebilirsiniz (Go 1.16+).
- Büyük asset setlerinde bir asset yöneticisi veya cache katmanı kullanmak iyi olur.

## Desteklenen Platformlar

- Masaüstü: Windows, macOS, Linux
- Web: WebAssembly (WASM) + WebGL
- Mobil: Android, iOS

Dağıtım notları:

- Web için WASM paketleme ve bir basit HTTP servis gerekir.
- Mobil için Android SDK/NDK ve iOS toolchain gereksinimlerini göz önünde bulundurun.

## Yardımcı Kütüphaneler ve Modüller

- Girdi: `github.com/hajimehoshi/ebiten/v2/inpututil` (tuş/klik edge tespiti)
- Yazı: `github.com/hajimehoshi/ebiten/v2/text` + `golang.org/x/image/font`
- Ses: `github.com/hajimehoshi/ebiten/v2/audio`
- UI: `github.com/blizzy78/ebitenui` (UI kontrolleri)
- ECS (Entity Component System): `github.com/yohamta/donburi`, `github.com/sedyh/mizu`
- Fizik: [jakecoffman/cp](https://github.com/jakecoffman/cp) (Chipmunk2D – 2D rigid body)
- Utility: `github.com/hajimehoshi/ebiten/v2/ebitenutil` (debug çizimler, görsel yükleme)

Topluluk listeleri ve örnekler:

- Awesome Ebiten/Ebitengine: [sedyh/awesome-ebiten](https://github.com/sedyh/awesome-ebiten)
- Örnekler: [hajimehoshi/ebiten/examples](https://github.com/hajimehoshi/ebiten/tree/main/examples)

## Kaynaklar

- Resmi site: [ebitengine.org](https://ebitengine.org/)
- GitHub repo: [hajimehoshi/ebiten](https://github.com/hajimehoshi/ebiten)
- Dokümantasyon: [Ebitengine Documents](https://ebitengine.org/en/documents/)
- Bizim oyun örnekler: [2D Games - Golang](https://github.com/DTVegaArchChapter/GameProgramming/tree/main/2d-games)

---

Ebitengine; Go'nun sadeliğini oyun geliştirmeye taşıyarak hızlı prototiplemeye ve küçük/orta ölçekli 2D projelere güçlü bir zemin sunar. Kısa sürede çalışan bir pencere açıp görsel çizebilir, daha sonra input, ses, sahne yönetimi ve ECS gibi katmanları ekleyerek oyununuzu büyütebilirsiniz.
