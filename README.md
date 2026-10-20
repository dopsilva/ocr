# Sea OCR

Biblioteca de **OCR (Optical Character Recognition)** escrita em **[Sea](https://github.com/dopsilva/sea)**, utilizando `libs/tensor` para aceleração de rede neural (MLP) e `libs/compress` para leitura e escrita nativa de imagens PNG.

## Recursos

- **100% em [Sea](https://github.com/dopsilva/sea)**: sem dependência de motores externos de OCR (como Tesseract).
- **Formatos de imagem suportados**:
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
  - Divisão de caracteres colados por vale de projeção vertical
  - Agrupamento adaptativo de palavras e cálculo de bounding boxes (`{x, y, w, h}`) em todos os níveis
- **Classificador Híbrido (Template Estrutural + Rede Neural MLP com `libs/tensor`)**:
  - Extrator de características de **224 dimensões** (`float_array`): grid normalizado 10×14, projeções horizontais/verticais, perfis de contorno externo, cruzamentos de traço e topologia (furos/Euler + razão de aspecto + posição relativa na linha)
  - Fonte bitmap 5×7 e glifos TrueType (`DejaVuSansMono` Regular e Bold) embutidos (73 classes: `A-Z`, `a-z`, `0-9` e pontuação)
  - Treinamento neural com `libs/tensor` (`matvec`, `rmsnorm`, `relu`, `softmax`, `adam`) e persistência de modelos em JSON (`save`/`load`)

## Estrutura dos Módulos

| Arquivo | Descrição |
|---------|-----------|
| [`ocr.sea`](ocr.sea) | Ponto de entrada principal (`ocr.read`, `ocr.recognize`, `ocr.new_engine`) |
| [`image.sea`](image.sea) | Decodificação, codificação e manipulação de imagens PNG, BMP, PNM e ASCII |
| [`preprocess.sea`](preprocess.sea) | Filtros de pré-processamento, binarização Otsu/adaptativa, morfologia e deskew |
| [`segment.sea`](segment.sea) | Segmentação de linhas, palavras e caracteres com bounding boxes |
| [`features.sea`](features.sea) | Extração de vetor de características de 224 dimensões (`float_array`) |
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
