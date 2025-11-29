<template>
	<div class="simulation-container">
		<!-- Canvas Area -->
		<div class="canvas-area">
			<canvas ref="canvas" width="800" height="600" class="main-canvas"></canvas>
		</div>
		
		<!-- Side Panel -->
		<div class="side-panel">
			<!-- Métricas -->
			<div class="metrics-panel">
				<h3> Métricas</h3>
				<div class="metric-item">
					<span class="metric-label">Tempo até captura:</span>
					<span class="metric-value">{{ elapsedTimeDisplay }}</span>
				</div>
				<div class="metric-item">
					<span class="metric-label">Capturas:</span>
					<span class="metric-value">{{ captures }}</span>
				</div>
				<div class="metric-item">
					<span class="metric-label">Tentativas:</span>
					<span class="metric-value">{{ totalAttempts }}</span>
				</div>
				<div class="metric-item">
					<span class="metric-label">Taxa de sucesso:</span>
					<span class="metric-value success-rate">{{ successRate }}%</span>
				</div>
				<div v-if="!targetDetected" class="lost-indicator">
					Alvo perdido!
				</div>
			</div>

			<!-- Controles -->
			<div class="control-panel">
				<h3> Controles</h3>
				<div class="control-group">
					<label class="control-label">
						Ligeirinho Velocidade: <span class="control-value">{{ targetSpeed }} px/frame</span>
						<input type="range" min="8" max="15" step="1" v-model.number="targetSpeed" @input="onTargetSpeedChange" class="control-slider" />
					</label>
				</div>
				<div class="control-group">
					<label class="control-label">
						Frajola Velocidade: <span class="control-value">{{ pursuerSpeed }} px/frame</span>
						<input type="range" min="6" max="12" step="1" v-model.number="pursuerSpeed" @input="onPursuerSpeedChange" class="control-slider" />
					</label>
				</div>
				<div class="control-group">
					<label class="control-label">
						Facilidade de Detecção: <span class="control-value">{{ detectionThreshold }}</span>
						<input type="range" min="10" max="100" step="1" v-model.number="detectionThreshold" @input="onDetectionChange" class="control-slider" />
					</label>
				</div>
				<div class="control-group">
					<label class="control-label">
						FPS: <span class="control-value">{{ fps }}</span>
						<input type="range" min="10" max="60" step="1" v-model.number="fps" @input="onFpsChange" class="control-slider" />
					</label>
				</div>
			</div>
		</div>
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
			lostCount: 0, // Contador de perdas do alvo
			elapsedTime: 0,
			lastCaptureTime: 0,
			// Controle de animação
			animationId: null,
			lastFrame: null,
			lostFrames: 0,
			lostLimit: 30, // Aumentado para tornar perda mais rara
		};
	},
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

		/**
		 * Atualiza lógica dos agentes, detecção, captura e métricas
		 * @param {number} dt - delta time em ms
		 */
		update(dt) {
			// Atualiza tempo
			this.elapsedTime += dt;
			
			// Move Ligeirinho
			this.target.x += this.target.dx * this.targetSpeed;
			this.target.y += this.target.dy * this.targetSpeed;
			
			// Detecção visual: alvo é detectado se estiver dentro do canvas
			this.detectTarget();
			
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
			// Verifica se o alvo está dentro das bordas do canvas
			const isInBounds = (
				this.target.x >= 0 && 
				this.target.x + this.target.size <= this.width &&
				this.target.y >= 0 && 
				this.target.y + this.target.size <= this.height
			);
			
			if (!isInBounds) {
				this.targetDetected = false;
				return;
			}
			
			// Simula dificuldade de detecção baseada na velocidade e sensibilidade
			const currentSpeed = Math.sqrt(
				Math.pow(this.target.dx * this.targetSpeed, 2) + 
				Math.pow(this.target.dy * this.targetSpeed, 2)
			);
			
			// Chance de perder o alvo baseada na velocidade vs sensibilidade
			const detectionDifficulty = currentSpeed / this.detectionThreshold;
			const lossChance = Math.max(0, detectionDifficulty - 0.5);
			
			// Aplica chance de perda apenas ocasionalmente
			if (Math.random() < lossChance * 0.02) { // 2% de chance por frame quando difícil
				this.lostFrames++;
				if (this.lostFrames > this.lostLimit) {
					this.targetDetected = false;
				}
			} else {
				this.targetDetected = true;
				this.lostFrames = 0;
			}
		},

		render() {
			// Limpa canvas
			this.ctx.clearRect(0, 0, this.width, this.height);
			
			// Desenha Ligeirinho (alvo)
			this.ctx.fillStyle = '#FFD700';
			this.ctx.fillRect(this.target.x, this.target.y, this.target.size, this.target.size);
			
			// Desenha Frajola (perseguidor)
			this.ctx.fillStyle = '#222';
			this.ctx.fillRect(this.pursuer.x, this.pursuer.y, this.pursuer.size, this.pursuer.size);
			
			// Indicação visual de detecção
			if (!this.targetDetected) {
				this.ctx.strokeStyle = 'red';
				this.ctx.lineWidth = 4;
				this.ctx.strokeRect(this.target.x, this.target.y, this.target.size, this.target.size);
			}
		},

		onTargetSpeedChange() {
			// Ajusta automaticamente a velocidade do Frajola para 70% da velocidade do alvo
			this.pursuerSpeed = Math.floor(this.targetSpeed * 0.7);
		},

		onPursuerSpeedChange() {
			// Calcula a velocidade do alvo baseada na velocidade do perseguidor (Frajola = 70% do alvo)
			this.targetSpeed = Math.ceil(this.pursuerSpeed / 0.7);
			
			// Garante que não ultrapasse os limites do slider do alvo
			if (this.targetSpeed > 15) {
				this.targetSpeed = 15;
				this.pursuerSpeed = Math.floor(this.targetSpeed * 0.7);
			}
		},

		onDetectionChange() {
			// Reset de frames perdidos ao mudar sensibilidade
			this.lostFrames = 0;
			this.targetDetected = true;
		},

		onFpsChange() {
			// Nada extra, mas pode ser usado para lógica futura
		},
	}
}
</script>

<style scoped>
.simulation-container {
	display: flex;
	gap: 20px;
	padding: 20px;
	background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
	min-height: 100vh;
	font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.canvas-area {
	flex-shrink: 0;
}

.main-canvas {
	border: 3px solid #2c3e50;
	background: #ecf0f1;
	border-radius: 8px;
	box-shadow: 0 4px 15px rgba(0,0,0,0.1);
}

.side-panel {
	width: 280px;
	display: flex;
	flex-direction: column;
	gap: 20px;
}

.metrics-panel {
	position: relative;
	background: #ffffff;
	border: 2px solid #3498db;
	border-radius: 12px;
	padding: 20px;
	box-shadow: 0 4px 15px rgba(52, 152, 219, 0.1);
	min-height: 180px; /* Altura fixa para evitar deslocamento */
}

.metrics-panel h3 {
	margin: 0 0 15px 0;
	color: #2c3e50;
	font-size: 18px;
	font-weight: 600;
}

.metric-item {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 8px 0;
	border-bottom: 1px solid #ecf0f1;
}

.metric-item:last-child {
	border-bottom: none;
}

.metric-label {
	color: #7f8c8d;
	font-weight: 500;
	font-size: 14px;
}

.metric-value {
	color: #2c3e50;
	font-weight: bold;
	font-size: 16px;
}

.success-rate {
	color: #27ae60;
	font-size: 18px;
}

.lost-indicator {
	position: absolute;
	bottom: 15px;
	left: 15px;
	right: 15px;
	background: #e74c3c;
	color: white;
	padding: 8px 12px;
	border-radius: 6px;
	font-weight: bold;
	text-align: center;
	animation: pulse 1s infinite;
	box-shadow: 0 2px 8px rgba(231, 76, 60, 0.3);
}

@keyframes pulse {
	0% { opacity: 1; }
	50% { opacity: 0.7; }
	100% { opacity: 1; }
}

.control-panel {
	background: #ffffff;
	border: 2px solid #e67e22;
	border-radius: 12px;
	padding: 20px;
	box-shadow: 0 4px 15px rgba(230, 126, 34, 0.1);
}

.control-panel h3 {
	margin: 0 0 15px 0;
	color: #2c3e50;
	font-size: 18px;
	font-weight: 600;
}

.control-group {
	margin-bottom: 15px;
}

.control-label {
	display: block;
	color: #2c3e50;
	font-weight: 500;
	font-size: 14px;
	line-height: 1.4;
	user-select: none;
	-webkit-user-select: none;
	-moz-user-select: none;
	cursor: default;
}

.control-value {
	color: #e67e22;
	font-weight: bold;
	float: right;
}

.control-slider {
	width: 100%;
	margin-top: 8px;
	height: 8px;
	border-radius: 4px;
	background: #ecf0f1;
	outline: none;
	appearance: none;
	-webkit-appearance: none;
	-moz-appearance: none;
	cursor: pointer;
	border: none;
}

.control-slider::-webkit-slider-track {
	width: 100%;
	height: 8px;
	cursor: pointer;
	background: #ecf0f1;
	border-radius: 4px;
	border: none;
}

.control-slider::-webkit-slider-thumb {
	-webkit-appearance: none;
	width: 20px;
	height: 20px;
	border-radius: 50%;
	background: #e67e22;
	cursor: pointer;
	border: 2px solid #ffffff;
	box-shadow: 0 2px 6px rgba(0,0,0,0.2);
	margin-top: -6px;
}

.control-slider::-moz-range-track {
	width: 100%;
	height: 8px;
	cursor: pointer;
	background: #ecf0f1;
	border-radius: 4px;
	border: none;
}

.control-slider::-moz-range-thumb {
	width: 20px;
	height: 20px;
	border-radius: 50%;
	background: #e67e22;
	cursor: pointer;
	border: 2px solid #ffffff;
	box-shadow: 0 2px 6px rgba(0,0,0,0.2);
	-moz-appearance: none;
}

.control-slider:focus {
	outline: 2px solid #3498db;
	outline-offset: 2px;
}

.control-slider:hover::-webkit-slider-thumb {
	background: #d35400;
	transform: scale(1.1);
	transition: all 0.2s ease;
}

.control-slider:hover::-moz-range-thumb {
	background: #d35400;
	transform: scale(1.1);
	transition: all 0.2s ease;
}

/* Hover effects */
.metrics-panel:hover {
	transform: translateY(-2px);
	transition: transform 0.2s ease;
}

.control-panel:hover {
	transform: translateY(-2px);
	transition: transform 0.2s ease;
}
</style>
