# Ficha 2 - Parte 2: Simulação de Visão Computacional

## Alunos:
- Adrian Modesto Lauzid
- Artur Vinícius Lima
- Gustavo dos Santos Silva
- Lucas Pereira de Souza

Este projeto é uma simulação interativa desenvolvida em Vue, para a disciplina de Visão Computacional. O objetivo é simular a perseguição de um alvo (Ligeirinho) por um perseguidor (Frajola), permitindo o ajuste de parâmetros e a visualização de métricas em tempo real.

## Funcionalidades

- **Canvas interativo:** Exibe a movimentação do alvo e do perseguidor.
- **Métricas em tempo real:** Tempo até captura, número de capturas, tentativas e taxa de sucesso.
- **Controles dinâmicos:** Ajuste de velocidade do alvo (Ligeirinho), perseguidor (Frajola), sensibilidade de detecção e FPS.
- **Indicação de alvo perdido:** Mensagem visual e destaque no canvas quando o alvo não é detectado.
- **Lógica de detecção:** Simula a dificuldade de detecção do alvo baseada em velocidade e sensibilidade.

## Estratégias de Perseguição

- **Perseguição Direta**: o perseguidor move-se diretamente em direção ao alvo enquanto ele estiver detectado.
- **Interceptação**: prediz a posição futura do alvo e tenta interceptá-lo; a predição considera as bordas do canvas e evita que o perseguidor saia da área visível.
- **Patrulhamento**: percorre pontos de patrulha; ao detectar o alvo, persegue por um tempo limitado e depois retorna à patrulha.
- **Busca Aleatória Inteligente**: combina movimento espiral/aleatório para explorar a área e persegue por curto período quando detecta o alvo.

O projeto coleta métricas por estratégia (capturas, tentativas, tempo médio e taxa de sucesso) para permitir comparações diretas entre modelos.

## Tecnologias Utilizadas

- [Vue](https://vuejs.org/) (Composition API)
- [Vite](https://vitejs.dev/) para build e desenvolvimento

## Estrutura do Projeto

- `src/App.vue`: Componente principal que renderiza o Canvas.
- `src/components/Canvas.vue`: Toda a lógica da simulação, controles e métricas.
- `public/`: Arquivos estáticos.
- `package.json`: Dependências e scripts.

## Como Executar

1. Instale as dependências:
	```bash
	npm install
	```
2. Rode o projeto em modo desenvolvimento:
	```bash
	npm run dev
	```
3. Acesse no navegador o endereço indicado (geralmente http://localhost:5173).

## Controles Disponíveis

- **Ligeirinho Velocidade:** Ajusta a velocidade do alvo.
- **Frajola Velocidade:** Ajusta a velocidade do perseguidor (limitado proporcionalmente ao alvo).
- **Sensibilidade de Detecção:** Quanto maior, mais fácil detectar o alvo.
- **FPS:** Controla a taxa de atualização da simulação.

## Métricas

- **Tempo até captura:** Tempo decorrido até o perseguidor capturar o alvo.
- **Capturas:** Quantidade de vezes que o perseguidor capturou o alvo.
- **Tentativas:** Soma de capturas e perdas do alvo.
- **Taxa de sucesso:** Percentual de capturas sobre tentativas.

## Observações

- O projeto é totalmente front-end e não requer backend.
- O código está comentado para facilitar o entendimento da lógica de simulação e detecção.
