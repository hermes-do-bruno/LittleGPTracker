# Plano de Implementação de Gráficos de Áudio (Futuro)

Este documento descreve as propostas de design, arquitetura e fórmulas matemáticas para renderização de gráficos em tempo real no **LittleGPTracker**.

---

## 1. EQ Curve (Curva de Resposta em Frequência) na `EqualizerView`

O objetivo é desenhar um gráfico visual representando a atenuação e ganho acumulado das 6 bandas do equalizador paramétrico (`EqPs`) na `EqualizerView`, localizada logo abaixo da visualização do instrumento.

### 1.1. Abordagem de Renderização
Como o backend do LGPT (`SDLGUIWindowImp`) suporta apenas desenho de texto e blocos sólidos (`DrawRect`), a curva será renderizada usando **barras verticais adjacentes de 1 pixel de largura** ou pequenos segmentos de blocos, agindo como um gráfico de linhas contínuo.

- **Eixo X (Frequência)**: Escala logarítmica representando a faixa audível de $20\text{ Hz}$ a $20\text{ kHz}$.
- **Eixo Y (Ganho em dB)**: Centrado verticalmente em $0\text{ dB}$, estendendo-se de $-15\text{ dB}$ a $+15\text{ dB}$ (ou o limite suportado pelas bandas do EQ).

### 1.2. Equações e Cálculo da Curva
Para cada pixel horizontal $x$ na largura reservada para o gráfico (por exemplo, 80 pixels):
1. **Mapear Pixel para Frequência ($f$)**:
   $$f(x) = f_{\text{min}} \cdot \left(\frac{f_{\text{max}}}{f_{\text{min}}}\right)^{\frac{x}{\text{width}}}$$
   Onde $f_{\text{min}} = 20$, $f_{\text{max}} = 20000$ e $\text{width}$ é a largura horizontal do gráfico em pixels.

2. **Calcular a Magnitude Acumulada ($H(f)$)**:
   A resposta de magnitude combinada de filtros biquad em cascata é o produto das magnitudes de cada banda individual. Em decibéis (dB), a resposta é a soma direta dos ganhos individuais de cada uma das 6 bandas:
   $$H_{\text{total}}(f)\text{ [dB]} = \sum_{b=1}^{6} H_b(f)\text{ [dB]}$$

   Para um filtro paramétrico do tipo *Peaking EQ*, a resposta de magnitude individual em dB de uma banda $b$ com frequência central $f_0$, ganho $G$ (linear, onde $G = 10^{\text{gain\_db}/20}$) e fator de qualidade $Q$ é calculada avaliando a função de transferência do biquad no domínio da frequência analógica aproximada ou digital bilinear:
   $$H_b(f) = 20 \log_{10} \left| \frac{s^2 + s \cdot \frac{G}{Q} + 1}{s^2 + s \cdot \frac{1}{Q \cdot G} + 1} \right| \quad \text{onde } s = j \frac{f}{f_0}$$

3. **Mapear Ganho em dB para Eixo Y (Pixel $y$)**:
   $$y(x) = y_{\text{centro}} - \left( H_{\text{total}}(f(x)) \cdot \text{escala\_y} \right)$$
   Onde $y_{\text{centro}}$ é a linha correspondente a $0\text{ dB}$ e $\text{escala\_y}$ é o fator de conversão de dB para pixels verticais.

4. **Desenho**:
   Para cada coluna $x$, desenha-se um pixel ou pequeno retângulo em $y(x)$.

---

## 2. WaveForm Display (Forma de Onda) na `InstrumentView`

O objetivo é prover um mini-osciloscópio / representação estática da forma de onda do sample carregado na aba de edição do instrumento (`InstrumentView`).

### 2.1. Abordagem de Renderização
A visualização do sample será posicionada horizontalmente ao lado das propriedades básicas do instrumento (ou em uma aba dedicada).

- **Eixo X**: Tempo / Amostras do sample (`Sample`).
- **Eixo Y**: Amplitude da onda (normalizada de $-1.0$ a $+1.0$).

### 2.2. Otimização por Downsampling
Como o sample possui dezenas de milhares de amostras e a tela do tracker possui poucos pixels de largura (ex: 120-160 pixels para o widget), renderizar todas as amostras diretamente seria ineficiente.

1. **Janelamento por Pixel (Min-Max Decimation)**:
   Para cada pixel horizontal $x$, calculamos um bloco correspondente de dados de áudio de tamanho $N$:
   $$N = \frac{\text{Tamanho Total do Sample (amostras)}}{\text{Largura do Gráfico em Pixels}}$$

2. Dentro da janela de amostras de $[x \cdot N$ até $(x+1) \cdot N]$, o algoritmo varre o buffer do sample e extrai os valores **Mínimo** ($A_{\text{min}}$) e **Máximo** ($A_{\text{max}}$).
3. Mapeia esses valores de amplitude para coordenadas de pixels verticais:
   $$y_{\text{top}} = y_{\text{centro}} - (A_{\text{max}} \cdot \text{altura\_max})$$
   $$y_{\text{bottom}} = y_{\text{centro}} - (A_{\text{min}} \cdot \text{altura\_max})$$
4. **Desenho**:
   Chama-se `DrawRect` desenhando uma barra vertical fina (1 pixel de largura) estendendo-se de $y_{\text{top}}$ até $y_{\text{bottom}}$. Isso gera uma forma de onda sólida e livre de aliasing, similar aos softwares de áudio profissionais.

---

## 3. Próximos Passos & Integração
- **Atualização**: Como calcular as curvas do EQ a cada frame é custoso para dispositivos embarcados (ex: consoles portáteis como Dingoo/GP2X), a curva do EQ deve ser **cacheadas** e recalculada apenas quando o usuário alterar interativamente o Ganho, Frequência ou Q de alguma banda na interface de edição.
- **Buffers Estáticos**: Para a forma de onda do sample, o downsampling deve ser calculado apenas **uma vez** durante o carregamento do áudio (`AssignSample`), guardando um array pequeno de `width` elementos contendo os valores de pico pré-calculados.
