// --- Canvas.vue ---
// Simulação de perseguição 2D: Frajola (perseguidor) e Ligeirinho (alvo)
// Componentes: detecção visual, spawn em bordas, captura, controles dinâmicos e métricas
	/**
	 * Reinicia o alvo (Ligeirinho) em uma borda aleatória com direção aleatória
	 */
	/**
	 * Reinicia o perseguidor (Frajola) no centro do canvas
	 */
	/**
	 * Loop principal de animação: atualiza lógica e renderiza frame
	 */
	/**
	 * Atualiza lógica dos agentes, detecção, captura e métricas
	 * @param {number} dt - delta time em ms
	 */
	/**
	 * Detecção visual do alvo usando diferença de quadros (simulada)
	 * Marca alvo como perdido se diferença abaixo do limiar por vários frames
	 */
	/**
	 * Renderiza agentes e overlays no canvas
	 */
	/**
	 * Handler: ao mudar velocidade do alvo
	 * Garante que Frajola não ultrapasse 70% da velocidade do alvo
	 */
	/**
	 * Handler: ao mudar velocidade do perseguidor
	 * Garante que Frajola não ultrapasse 70% da velocidade do alvo
	 */
	/**
	 * Handler: ao mudar sensibilidade de detecção
	 * Reseta contagem de frames perdidos
	 */
	/**
	 * Handler: ao mudar FPS
	 * (Reservado para lógica futura)
	 */
<template>
	<div class="canvas-container">
		<canvas ref="canvas" width="800" height="600" class="main-canvas"></canvas>
		<!-- Overlay de métricas -->
		<div class="metrics-overlay">
			<div>Tempo até captura: <b>{{ elapsedTimeDisplay }}</b></div>
			<div>Capturas: <b>{{ captures }}</b></div>
			<div>Tentativas: <b>{{ totalAttempts }}</b></div>
			<div>Taxa de sucesso: <b>{{ successRate }}%</b></div>
			<div v-if="!targetDetected" class="lost-indicator">Alvo perdido!</div>
		</div>
			computed: {
				elapsedTimeDisplay() {
					return (this.elapsedTime / 1000).toFixed(2) + 's';
				},
				totalAttempts() {
					// Tentativas = capturas + perdas (quando alvo é perdido)
					return this.captures + this.lostCount;
				},
				successRate() {
					if (this.totalAttempts === 0) return 100;
					return ((this.captures / this.totalAttempts) * 100).toFixed(1);
				}
			},
			lostCount: 0, // Contador de perdas do alvo
		<!-- Painel de controle -->
		<div class="control-panel">
			<label>
				Ligeirinho Velocidade: <b>{{ targetSpeed }} px/frame</b>
				<input type="range" min="8" max="15" step="1" v-model.number="targetSpeed" @input="onTargetSpeedChange" />
			</label>
			<label>
				Frajola Velocidade: <b>{{ pursuerSpeed }} px/frame</b>
				<input type="range" min="6" max="12" step="1" v-model.number="pursuerSpeed" @input="onPursuerSpeedChange" />
			</label>
			<label>
				Sensibilidade Detecção: <b>{{ detectionThreshold }}</b>
				<input type="range" min="10" max="100" step="1" v-model.number="detectionThreshold" @input="onDetectionChange" />
			</label>
			<label>
				FPS: <b>{{ fps }}</b>
				<input type="range" min="10" max="60" step="1" v-model.number="fps" @input="onFpsChange" />
			</label>
		</div>
			onTargetSpeedChange() {
				// Garante que Frajola não ultrapasse 70% da velocidade do alvo
				if (this.pursuerSpeed > this.targetSpeed * 0.7) {
					this.pursuerSpeed = Math.floor(this.targetSpeed * 0.7);
				}
			},
			onPursuerSpeedChange() {
				// Garante que Frajola não ultrapasse 70% da velocidade do alvo
				if (this.pursuerSpeed > this.targetSpeed * 0.7) {
					this.pursuerSpeed = Math.floor(this.targetSpeed * 0.7);
				}
			},
			onDetectionChange() {
				// Reset de frames perdidos ao mudar sensibilidade
				this.lostFrames = 0;
			},
			onFpsChange() {
				// Nada extra, mas pode ser usado para lógica futura
			},
	</div>
</template>

<script>
// Utilidades matemáticas
function randomEdgePosition(width, height, size) {
	// Retorna {x, y, dx, dy} para spawn em borda e direção aleatória
	const edge = Math.floor(Math.random() * 4);
	let x, y, dx, dy, angle;
	angle = Math.random() * Math.PI * 2;
	switch (edge) {
		case 0: // Topo
			x = Math.random() * (width - size);
			y = 0;
			break;
		case 1: // Direita
			x = width - size;
			y = Math.random() * (height - size);
			break;
		case 2: // Base
			x = Math.random() * (width - size);
			y = height - size;
			break;
		case 3: // Esquerda
			x = 0;
			y = Math.random() * (height - size);
			break;
	}
	dx = Math.cos(angle);
	dy = Math.sin(angle);
	return { x, y, dx, dy };
}

function distance(ax, ay, bx, by) {
	return Math.sqrt((ax - bx) ** 2 + (ay - by) ** 2);
}

export default {
	name: 'Canvas',
	data() {
		return {
			canvas: null,
			ctx: null,
			width: 800,
			height: 600,
			// Agentes
			target: { x: 0, y: 0, dx: 1, dy: 0, size: 20 },
			pursuer: { x: 400 - 22.5, y: 300 - 22.5, size: 45 },
			// Configurações dinâmicas
			targetSpeed: 10,
			pursuerSpeed: 7,
			detectionThreshold: 30,
			fps: 30,
			// Estado de detecção
			targetDetected: true,
			// Métricas
			captures: 0,
			elapsedTime: 0,
			lastCaptureTime: 0,
			successRate: 100,
			// Controle de animação
			animationId: null,
			lastFrame: null,
			prevFrameData: null,
			lostFrames: 0,
			lostLimit: 10,
		};
	},
	computed: {
		elapsedTimeDisplay() {
			return (this.elapsedTime / 1000).toFixed(2) + 's';
		}
	},
	mounted() {
		this.canvas = this.$refs.canvas;
		this.ctx = this.canvas.getContext('2d');
		this.resetTarget();
		this.resetPursuer();
		this.lastFrame = performance.now();
		this.animate();
	},
	beforeDestroy() {
		cancelAnimationFrame(this.animationId);
	},
	methods: {
		resetTarget() {
			// Spawn Ligeirinho em borda aleatória
			const pos = randomEdgePosition(this.width, this.height, this.target.size);
			this.target.x = pos.x;
			this.target.y = pos.y;
			this.target.dx = pos.dx;
			this.target.dy = pos.dy;
		},
		resetPursuer() {
			// Frajola no centro
			this.pursuer.x = this.width / 2 - this.pursuer.size / 2;
			this.pursuer.y = this.height / 2 - this.pursuer.size / 2;
		},
		animate() {
			this.animationId = requestAnimationFrame(this.animate);
			const now = performance.now();
			const dt = now - this.lastFrame;
			if (dt < 1000 / this.fps) return;
			this.lastFrame = now;
			this.update(dt);
			this.render();
		},
		update(dt) {
			// Atualiza tempo
			this.elapsedTime += dt;
			// Move Ligeirinho
			this.target.x += this.target.dx * this.targetSpeed;
			this.target.y += this.target.dy * this.targetSpeed;
			// Gerenciamento de bordas: se saiu, respawna
			if (
				this.target.x < -this.target.size ||
				this.target.x > this.width ||
				this.target.y < -this.target.size ||
				this.target.y > this.height
			) {
				// Se o alvo saiu do canvas, conta como perda
				this.lostCount++;
				this.resetTarget();
				this.elapsedTime = 0;
				this.targetDetected = true;
				this.lostFrames = 0;
				return;
			}
			// Detecção visual (diferença de quadros simplificada)
			this.detectTarget();
			// Move Frajola se alvo detectado
			if (this.targetDetected) {
				const px = this.pursuer.x + this.pursuer.size / 2;
				const py = this.pursuer.y + this.pursuer.size / 2;
				const tx = this.target.x + this.target.size / 2;
				const ty = this.target.y + this.target.size / 2;
				const angle = Math.atan2(ty - py, tx - px);
				this.pursuer.x += Math.cos(angle) * this.pursuerSpeed;
				this.pursuer.y += Math.sin(angle) * this.pursuerSpeed;
			}
			// Captura
			const px = this.pursuer.x + this.pursuer.size / 2;
			const py = this.pursuer.y + this.pursuer.size / 2;
			const tx = this.target.x + this.target.size / 2;
			const ty = this.target.y + this.target.size / 2;
			if (distance(px, py, tx, ty) < 20) {
				// Captura bem-sucedida
				this.captures++;
				this.lastCaptureTime = this.elapsedTime;
				this.resetTarget();
				this.resetPursuer();
				this.elapsedTime = 0;
				this.targetDetected = true;
				this.lostFrames = 0;
			}
		},
		detectTarget() {
			// Simulação de detecção visual: diferença de quadros (simplificada)
			// Captura frame atual
			const frame = this.ctx.getImageData(0, 0, this.width, this.height);
			if (this.prevFrameData) {
				let diff = 0;
				for (let i = 0; i < frame.data.length; i += 4) {
					diff += Math.abs(frame.data[i] - this.prevFrameData[i]);
				}
				// Se diferença abaixo do limiar, considera alvo perdido
				if (diff / (this.width * this.height) < this.detectionThreshold) {
					this.lostFrames++;
					if (this.lostFrames > this.lostLimit) {
						this.targetDetected = false;
					}
				} else {
					this.targetDetected = true;
					this.lostFrames = 0;
				}
			}
			this.prevFrameData = frame.data.slice();
		},
		render() {
			// Limpa canvas
			this.ctx.clearRect(0, 0, this.width, this.height);
			// Desenha Ligeirinho
			this.ctx.fillStyle = '#FFD700';
			this.ctx.fillRect(this.target.x, this.target.y, this.target.size, this.target.size);
			// Desenha Frajola
			this.ctx.fillStyle = '#222';
			this.ctx.fillRect(this.pursuer.x, this.pursuer.y, this.pursuer.size, this.pursuer.size);
			// Indicação visual de detecção
			if (!this.targetDetected) {
				this.ctx.strokeStyle = 'red';
				this.ctx.lineWidth = 4;
				this.ctx.strokeRect(this.target.x, this.target.y, this.target.size, this.target.size);
			}
		},
	}
}
</script>

<style scoped>
.canvas-container {
	position: relative;
	width: 800px;
	height: 600px;
}
.main-canvas {
	border: 2px solid #333;
	background: #e0e0e0;
}
.metrics-overlay {
	position: absolute;
	top: 10px;
	left: 10px;
	background: rgba(255,255,255,0.8);
	padding: 8px 16px;
	border-radius: 8px;
	font-size: 16px;
	z-index: 2;
}
.lost-indicator {
	color: red;
	font-weight: bold;
}
.control-panel {
	position: absolute;
	bottom: 10px;
	left: 10px;
	background: rgba(255,255,255,0.9);
	padding: 10px 18px;
	border-radius: 8px;
	z-index: 2;
	display: flex;
	flex-direction: column;
	gap: 8px;
	font-size: 15px;
}
input[type="range"] {
	width: 120px;
	margin-left: 8px;
}
</style>
