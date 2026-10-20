# Sea OCR

Biblioteca de **OCR (Optical Character Recognition)** escrita em **[Sea](https://github.com/dopsilva/sea)**, utilizando `libs/tensor` para aceleração de rede neural (MLP) e `libs/image` (com `libs/compress` para PNG) para leitura e escrita de imagens.

## Recursos

- **100% em [Sea](https://github.com/dopsilva/sea)**: sem dependência de motores externos de OCR (como Tesseract).
- **Formatos de imagem suportados** (via [`libs/image`](../sea_vm/libs/image/image.sea), compartilhada com o Sea QRCode):
  - **PNG** (8-bit Grayscale, RGB, RGBA, Palette + filtros Paeth, Sub, Up, Average)
  - **BMP** (1, 8, 24 e 32 bits)
  - **PGM / PPM / PBM** (`P1` a `P6`)
  - **ASCII Art** (`image.from_ascii` / `img:to_ascii`)
- **Pipeline de pré-processamento completo**:
  - `grayscale`, `invert`, `auto_invert` (detecção automática de fundo escuro), `normalize_contrast`
  - Binarização global de **Otsu** (`otsu_threshold`, `binarize`) e **binarização adaptativa** rápida via imagem integral (`adaptive_threshold`)
  - Remoção de ruído (`denoise` por filtro de mediana 3×3 ou remoção de speckles isolados)
  - Operações morfológicas (`dilate`, `erode`, `morph_open`, `morph_close`)
  - Geometria e alinhamento (`crop`, `trim`, `resize` bilinear/box, `rotate`, `estimate_skew` e `deskew` automático)
- **Segmentação hierárquica (Linhas → Palavras → Caracteres)**:
  - Rotulagem de componentes conexos (8-conectividade)
  - Fusão automática de pontos e diacríticos (`i`, `j`, `!`, `?`, `:`, `=`)
  - Segmentação guiada pelo reconhecimento, palavra a palavra: peças largas lidas com pouca confiança (`20` lido como `a`, `IN` como `m`) são cortadas em fatias nos vales de tinta e no meio, e uma programação dinâmica reagrupa as fatias (até 4 por caractere) do jeito cujos caracteres são lidos com mais confiança (soma de largura × confiança, menos um custo por caractere). Fatias de uma peça podem se juntar a uma vizinha onde um traço fino se quebrou (`00` lido como `(` + `m`); traços soltos e pontuação nascidos de um corte custam mais (são o que sobra de cortar m, n, u, w), salvo um `1` e um hífen encostado num número (opção `split_touching = false` desliga os cortes)
  - Texto pequeno (linhas com menos de 20 px) é ampliado em escala de cinza (bilinear, `preprocess.upscale`) antes da binarização; as caixas voltam às coordenadas da imagem original (opção `upscale`: o fator, ou `1` para nunca ampliar)
  - Agrupamento de palavras pelo limiar de Otsu sobre as distâncias entre caracteres (duas classes: entre letras e entre palavras) e cálculo de bounding boxes (`{x, y, w, h}`) em todos os níveis
- **Português**: letras acentuadas (`á à â ã é ê í ó ô õ ú ç` e maiúsculas, `º ª`) — uma marca acima da letra é lida à parte (ponto, agudo, grave, circunflexo, til, trema) e composta com a letra (`accents.sea`); acentos encostados na letra são reconhecidos pelo protótipo da letra acentuada
- **Correção por contexto** (`context.sea`), por linha: maiúscula/minúscula de letras de mesmo desenho (`c o s u v w x z`, `ç` e as vogais acentuadas `ó ô õ ú`...) pela altura do topo da letra acima da linha de base (sem a cedilha nem o acento) comparada à altura-x e à das maiúsculas da linha; em trechos numéricos as letras parecidas viram dígitos (`O→0`, `l/I→1`, `S→5`…) e o inverso em palavras; `I`/`l` pela caixa da palavra; `.`, `-` e `_` pela geometria
- **Classificador Híbrido (Template Estrutural + Rede Neural MLP com `libs/tensor`)**:
  - Extrator de características de **384 dimensões** (`float_array`): grid normalizado 10×14, projeções horizontais/verticais, perfis de contorno externo, cruzamentos de traço, topologia (furos/Euler + razão de aspecto + posição relativa na linha) e **direções do traço** (gradientes de Sobel de uma cópia 16×20 em 8 orientações, somados em 4×5 zonas com cada pixel dividido entre as 4 zonas mais próximas), que toleram melhor um traço um pouco mais alto ou mais fino que o aprendido (o `e` de barra alta do FreeSerif não é mais lido como `c`)
  - Fonte bitmap 5×7 e glifos TrueType embutidos: `DejaVuSansMono` Regular e Bold, e em [`glyphs.sea`](glyphs.sea) (gerado por [`tools/gen_glyphs.sea`](tools/gen_glyphs.sea) com o rasterizador TrueType em Sea puro [`libs/truetype`](../sea_vm/libs/truetype/truetype.sea), sem Python) 17 faces: DejaVu Sans/Serif/Mono (e Sans Bold, Serif Bold, Serif Condensed), Liberation Sans/Serif/Mono (e Sans Bold, Serif Bold), Noto Sans/Serif (e Serif Bold, Serif Condensed), Gentium Basic e Ubuntu — 133 classes: `A-Z`, `a-z`, `0-9`, pontuação, `$ % & * # @ ; ' " _` e as letras acentuadas
  - Cada glifo é gravado com quatro níveis de cobertura e aprendido em três espessuras de traço (fina, normal e grossa; `model.STROKES`): uma fonte um pouco mais leve ou mais pesada que as do treino ainda encontra as suas formas. Várias serifadas no treino ensinam o que uma serifada desconhecida tem em comum com elas
  - Os protótipos do modelo padrão ficam em cache em `~/.cache/sea-ocr/` (ou `$XDG_CACHE_HOME`): a primeira execução leva ~8 s para montá-los, as seguintes ~0,4 s. A chave é o hash dos glifos e das opções, e ao carregar os protótipos da fonte bitmap são refeitos e comparados (uma mudança no código das características refaz o cache, e os caches antigos são apagados). `model.new({cache = false})` não usa o cache
  - A comparação com os protótipos é um produto matriz-vetor (`tensor.matvec`) contra a matriz dos protótipos normalizados, ~17× mais rápida que o cosseno em Sea
  - Acerto por caractere em texto renderizado (11 fontes, 4 tamanhos, texto em inglês e em português): ~97% em 24–36 px, ~90% em 12 px, ~95% na média (texto do PIL em inglês; ~94% em português), ~85% na serifada mais difícil fora do treino (FreeSerif)
  - Treinamento neural com `libs/tensor` (`matvec`, `rmsnorm`, `relu`, `softmax`, `adam`) e persistência de modelos em JSON (`save`/`load`)

## Estrutura dos Módulos

| Arquivo | Descrição |
|---------|-----------|
| [`ocr.sea`](ocr.sea) | Ponto de entrada principal (`ocr.read`, `ocr.recognize`, `ocr.new_engine`) |
| [`libs/image`](../sea_vm/libs/image/image.sea) | (no sea_vm) Decodificação, codificação e manipulação de imagens PNG, BMP, PNM e ASCII; `ocr.image` a expõe |
| [`preprocess.sea`](preprocess.sea) | Filtros de pré-processamento, binarização Otsu/adaptativa, morfologia e deskew |
| [`segment.sea`](segment.sea) | Segmentação de linhas, palavras e caracteres com bounding boxes (e as peças de cada caractere) |
| [`accents.sea`](accents.sea) | Letra + marca (ponto, agudo, grave, circunflexo, til, trema) lidas à parte e compostas |
| [`context.sea`](context.sea) | Correções por linha: caixa pela altura, dígitos × letras por trecho, `I`/`l`, pontuação pela geometria |
| [`ttf.sea`](ttf.sea) | Glifos de uma fonte TrueType para o modelo (usado por `tools/gen_glyphs.sea` e `register_ttf`) |
| [`glyphs.sea`](glyphs.sea) | Glifos pré-rasterizados de 17 faces, com acentos e quatro níveis de cobertura (gerado por `tools/gen_glyphs.sea`) |
| [`features.sea`](features.sea) | Extração de vetor de características de 384 dimensões (`float_array`), com as direções do traço |
| [`font.sea`](font.sea) | Fonte bitmap 5×7 embutida, registro de fontes customizadas e renderizador de texto |
| [`ttf_font.sea`](ttf_font.sea) | Glifos TrueType (`DejaVuSansMono` Regular & Bold) pré-rasterizados em Sea |
| [`model.sea`](model.sea) | Classificador híbrido (Template + MLP via `libs/tensor`), treinamento e serialização |
| [`ocr_test.sea`](ocr_test.sea) | Suíte de testes automatizados (`sea test`) |

## Uso Rápido

```sea
local ocr = require("./ocr")

-- 1. Extração simples de texto (de arquivo PNG/BMP/PGM ou objeto Image)
local text = ocr.read("examples/recibo.png")?
print(text)

-- 2. Reconhecimento detalhado com bounding boxes e score de confiança
local res = ocr.recognize("examples/recibo.png")?
printf("Texto:\n%s\nConfiança geral: %.2f%%\n", res.text, res.confidence * 100)
for _, line in res.lines do
    for _, word in line.words do
        printf("Palavra '%s' em (x=%d, y=%d, w=%d, h=%d) [%.1f%%]\n",
            word.text, word.bbox.x, word.bbox.y, word.bbox.w, word.bbox.h, word.confidence * 100)
    end
end

-- 3. Pipeline explícito de pré-processamento (fundo escuro + inclinação)
local img = ocr.image.load("examples/dark_plate.png")?
local clean = ocr.preprocess.auto_invert(img)
clean = ocr.preprocess.binarize(clean)
clean = ocr.preprocess.denoise(clean, {method = "speckle"})
local deskewed, angle = ocr.preprocess.deskew(clean)
printf("Ângulo corrigido: %.1f° -> Texto: %s\n", angle, ocr.read(deskewed)?)

-- 4. Treinamento neural MLP com libs/tensor e fontes customizadas
local eng = ocr.new_engine()
eng:train({epochs = 30, lr = 0.015})?
print(eng:read("examples/hello_sea.png", {mode = "hybrid"})?)
eng:save_model("modelo_ocr.json")?

-- 5. A fonte dos documentos que serão lidos: o engine aprende os glifos dela
-- (desenhados por ttf.sea / libs/truetype, nas três espessuras de traço)
local recibos = ocr.new_engine()
recibos:register_ttf("/usr/share/fonts/truetype/freefont/FreeSerif.ttf")?
print(recibos:read("examples/recibo.png")?)
```

## Executando os Testes e Exemplos

Requer a VM da linguagem **[Sea](https://github.com/dopsilva/sea)** compilada (com `libs/compress` e `libs/tensor`):

```bash
# Rodar a suíte de testes
sea test -v

# Ler uma imagem PNG real pela linha de comando
sea examples/read_png.sea examples/recibo.png
sea examples/read_png.sea examples/hello_sea.png
sea examples/read_png.sea examples/dark_plate.png

# Rodar a demonstração completa
sea examples/demo.sea
```
