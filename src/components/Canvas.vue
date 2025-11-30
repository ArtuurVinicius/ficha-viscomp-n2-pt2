<template>
	<div class="simulation-container">
		<!-- Canvas Area -->
		<div class="canvas-area">
			<canvas ref="canvas" width="800" height="600" class="main-canvas"></canvas>
			<!-- Indicador movido para ficar logo abaixo do canvas -->
			<div v-if="!targetDetected" class="lost-indicator">
				Alvo perdido!
			</div>
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
				<div class="strategy-stats">
					<h4>{{ pursuitStrategy === 'direct' ? 'Perseguição Direta' : 
						pursuitStrategy === 'intercept' ? 'Interceptação' : 
						pursuitStrategy === 'patrol' ? 'Patrulhamento' : 'Busca Aleatória' }}</h4>
					<div class="mini-metric">
						<span>Capturas: {{ currentStrategyStats.captures }}</span>
						<span>Taxa: {{ currentStrategyStats.successRate }}%</span>
					</div>
					<div class="mini-metric">
						<span>Tentativas: {{ currentStrategyStats.attempts }}</span>
						<span>Tempo médio: {{ currentStrategyStats.avgTime }}s</span>
					</div>
				</div>
			</div>
			<!-- Controles -->
			<div class="control-panel">
				<h3> Controles</h3>
				<div class="control-group">
					<label class="control-label">
						Estratégia de Perseguição:
						<select v-model="pursuitStrategy" @change="onStrategyChange" class="control-select">
							<option value="direct">Perseguição Direta</option>
							<option value="intercept">Interceptação</option>
							<option value="patrol">Patrulhamento</option>
							<option value="random">Busca Aleatória</option>
						</select>
					</label>
				</div>
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
						Sensibilidade de Detecção: <span class="control-value">{{ detectionThreshold }}</span>
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
			pursuitStrategy: 'direct', // Estratégia atual
			// Variáveis auxiliares para diferentes estratégias
			patrolPoints: [],
			currentPatrolIndex: 0,
			lastKnownPosition: { x: 0, y: 0 },
			randomDirection: { dx: 1, dy: 0 },
			chaseStartTime: null,
			searchChaseStart: null,
			spiralAngle: 0,
			chaseStartTime: null,
			searchChaseStart: null,
			spiralAngle: 0,
			chaseStartTime: null,
			searchChaseStart: null,
			spiralAngle: 0,
			// Estado de detecção
			targetDetected: true,
			// Métricas
			captures: 0,
			lostCount: 0, // Contador de perdas do alvo
			elapsedTime: 0,
			lastCaptureTime: 0,
			// Métricas por estratégia para comparação
			strategyMetrics: {
				direct: { captures: 0, attempts: 0, totalTime: 0 },
				intercept: { captures: 0, attempts: 0, totalTime: 0 },
				patrol: { captures: 0, attempts: 0, totalTime: 0 },
				random: { captures: 0, attempts: 0, totalTime: 0 }
			},
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
		},
		currentStrategyStats() {
			const stats = this.strategyMetrics[this.pursuitStrategy];
			const successRate = stats.attempts === 0 ? 0 : ((stats.captures / stats.attempts) * 100).toFixed(1);
			const avgTime = stats.captures === 0 ? 0 : (stats.totalTime / stats.captures / 1000).toFixed(2);
			return {
				captures: stats.captures,
				attempts: stats.attempts,
				successRate,
				avgTime
			};
		}
	},
	mounted() {
		this.canvas = this.$refs.canvas;
		this.ctx = this.canvas.getContext('2d');
		this.resetTarget();
		this.resetPursuer();
		this.initializeStrategy();
		this.lastFrame = performance.now();
		this.animate();
	},
	beforeDestroy() {
		cancelAnimationFrame(this.animationId);
	},
	methods: {
		resetTarget() {
			// Frajola sempre no centro
			const pursuerCenterX = this.pursuer.x + this.pursuer.size / 2;
			const pursuerCenterY = this.pursuer.y + this.pursuer.size / 2;

			// Ligeirinho nasce em uma das bordas, mas nunca perto do Frajola
			// Calcula as 4 bordas possíveis
			const borders = [
				{ x: 0, y: Math.random() * (this.height - this.target.size), dx: 1, dy: 0 }, // esquerda
				{ x: this.width - this.target.size, y: Math.random() * (this.height - this.target.size), dx: -1, dy: 0 }, // direita
				{ x: Math.random() * (this.width - this.target.size), y: 0, dx: 0, dy: 1 }, // topo
				{ x: Math.random() * (this.width - this.target.size), y: this.height - this.target.size, dx: 0, dy: -1 } // base
			];

			// Filtra bordas que estão longe o suficiente do Frajola (mínimo 200px)
			const minDist = 200;
			const farBorders = borders.filter(b => {
				const bx = b.x + this.target.size / 2;
				const by = b.y + this.target.size / 2;
				return distance(bx, by, pursuerCenterX, pursuerCenterY) > minDist;
			});
			// Se todas estão perto, usa todas (fallback)
			const options = farBorders.length > 0 ? farBorders : borders;
			// Escolhe uma borda aleatória
			const spawn = options[Math.floor(Math.random() * options.length)];
			this.target.x = spawn.x;
			this.target.y = spawn.y;

			// Calcula direção para cruzar o centro, mas com leve desvio para "desviar" do Frajola
			const centerX = this.width / 2;
			const centerY = this.height / 2;
			let angleToCenter = Math.atan2(centerY - (this.target.y + this.target.size / 2), centerX - (this.target.x + this.target.size / 2));
			// Adiciona desvio de até ±30 graus para simular "desvio" do Frajola
			const deviation = (Math.random() - 0.5) * (Math.PI / 3); // ±30°
			angleToCenter += deviation;
			this.target.dx = Math.cos(angleToCenter);
			this.target.dy = Math.sin(angleToCenter);
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
			// Delay após captura/perda
			if (this._waitingTransition) return;

			// Atualiza tempo
			this.elapsedTime += dt;

			// Move Ligeirinho
			this.target.x += this.target.dx * this.targetSpeed;
			this.target.y += this.target.dy * this.targetSpeed;

			// Detecção visual: alvo é detectado se estiver dentro do canvas
			this.detectTarget();

			// Gerenciamento de bordas: se saiu, respawna com delay
			if (
				this.target.x < -this.target.size ||
				this.target.x > this.width ||
				this.target.y < -this.target.size ||
				this.target.y > this.height
				) {
					this.lostCount++;
					// Atualiza tentativas da estratégia atual (perda)
					this.strategyMetrics[this.pursuitStrategy].attempts++;
					this._waitingTransition = true;
					setTimeout(() => {
						this.resetTarget();
						this.elapsedTime = 0;
						this.targetDetected = true;
						this.lostFrames = 0;
						this._waitingTransition = false;
					}, 1000);
					return;
				}			// Move Frajola baseado na estratégia escolhida
			this.movePursuer();

			// Captura
			const px = this.pursuer.x + this.pursuer.size / 2;
			const py = this.pursuer.y + this.pursuer.size / 2;
			const tx = this.target.x + this.target.size / 2;
			const ty = this.target.y + this.target.size / 2;
			if (distance(px, py, tx, ty) < 20) {
				this.captures++;
				this.lastCaptureTime = this.elapsedTime;
				// Atualiza métricas da estratégia atual
				this.strategyMetrics[this.pursuitStrategy].captures++;
				this.strategyMetrics[this.pursuitStrategy].attempts++;
				this.strategyMetrics[this.pursuitStrategy].totalTime += this.elapsedTime;
				this._waitingTransition = true;
				setTimeout(() => {
					this.resetTarget();
					this.resetPursuer();
					this.elapsedTime = 0;
					this.targetDetected = true;
					this.lostFrames = 0;
					this._waitingTransition = false;
				}, 1000);
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
			// Novo cálculo: quanto menor o threshold, maior a chance de perder
			// Quanto maior o threshold, menor a chance de perder
			// Usar função exponencial para efeito perceptível
			// detectionThreshold: 10 (difícil), 100 (fácil)
			// lossChance varia de ~0.5 (difícil) até ~0.001 (fácil)
			const minThreshold = 10;
			const maxThreshold = 100;
			// Normaliza threshold para [0,1] (0 = difícil, 1 = fácil)
			const norm = (this.detectionThreshold - minThreshold) / (maxThreshold - minThreshold);
			// Inverte: 0 = fácil perder, 1 = difícil perder
			const difficulty = 1 - norm;
			// Base loss chance: depende da velocidade e da dificuldade
			// Para valores baixos de threshold, lossChance é alta (~5% por frame)
			// Para valores altos, lossChance é quase zero
			let lossChance = Math.min(1, (currentSpeed / 15) * (0.05 + 0.25 * difficulty * difficulty));
			// Para threshold máximo, lossChance ~0.005; para mínimo, ~0.05
			if (Math.random() < lossChance) {
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

		onStrategyChange() {
			// Reinicializa variáveis específicas da estratégia
			this.initializeStrategy();
		},

		initializeStrategy() {
			switch (this.pursuitStrategy) {
				case 'patrol':
					// Define pontos de patrulhamento
					this.patrolPoints = [
						{ x: 100, y: 100 },
						{ x: 700, y: 100 },
						{ x: 700, y: 500 },
						{ x: 100, y: 500 }
					];
					this.currentPatrolIndex = 0;
					break;
				case 'random':
					// Direção aleatória inicial
					const angle = Math.random() * Math.PI * 2;
					this.randomDirection = { dx: Math.cos(angle), dy: Math.sin(angle) };
					break;
			}
		},

		movePursuer() {
			const px = this.pursuer.x + this.pursuer.size / 2;
			const py = this.pursuer.y + this.pursuer.size / 2;
			let moveX = 0, moveY = 0;

			switch (this.pursuitStrategy) {
				case 'direct':
					moveX = this.directPursuit(px, py).dx;
					moveY = this.directPursuit(px, py).dy;
					break;
				case 'intercept':
					moveX = this.interceptPursuit(px, py).dx;
					moveY = this.interceptPursuit(px, py).dy;
					break;
				case 'patrol':
					moveX = this.patrolPursuit(px, py).dx;
					moveY = this.patrolPursuit(px, py).dy;
					break;
				case 'random':
					moveX = this.randomPursuit(px, py).dx;
					moveY = this.randomPursuit(px, py).dy;
					break;
			}

			this.pursuer.x += moveX * this.pursuerSpeed;
			this.pursuer.y += moveY * this.pursuerSpeed;
		},

		directPursuit(px, py) {
			// Perseguição direta: move em direção ao alvo se detectado
			if (!this.targetDetected) return { dx: 0, dy: 0 };
			
			const tx = this.target.x + this.target.size / 2;
			const ty = this.target.y + this.target.size / 2;
			const angle = Math.atan2(ty - py, tx - px);
			return { dx: Math.cos(angle), dy: Math.sin(angle) };
		},

	interceptPursuit(px, py) {
		// Interceptação: prediz onde o alvo estará considerando bordas
		if (!this.targetDetected) {
			// Se perdeu o alvo, vai para última posição conhecida
			const angle = Math.atan2(this.lastKnownPosition.y - py, this.lastKnownPosition.x - px);
			return this.constrainMovement(px, py, Math.cos(angle), Math.sin(angle));
		}

		// Atualiza última posição conhecida
		this.lastKnownPosition.x = this.target.x + this.target.size / 2;
		this.lastKnownPosition.y = this.target.y + this.target.size / 2;

		// Prediz posição futura do alvo com múltiplos cenários
		const predictionTime = 20;
		let futureX = this.target.x + this.target.dx * this.targetSpeed * predictionTime;
		let futureY = this.target.y + this.target.dy * this.targetSpeed * predictionTime;

		// Ajusta predição se alvo sair das bordas
		futureX = Math.max(0, Math.min(futureX, this.width - this.target.size));
		futureY = Math.max(0, Math.min(futureY, this.height - this.target.size));

		const angle = Math.atan2(futureY - py, futureX - px);
		return this.constrainMovement(px, py, Math.cos(angle), Math.sin(angle));
	},

	patrolPursuit(px, py) {
		// Patrulhamento: movimento sistemático em área, busca ativa
		const currentPoint = this.patrolPoints[this.currentPatrolIndex];
		const distToPoint = distance(px, py, currentPoint.x, currentPoint.y);

		// Se detectou alvo, persegue por tempo limitado antes de voltar à patrulha
		if (this.targetDetected) {
			if (!this.chaseStartTime) {
				this.chaseStartTime = Date.now();
			}
			// Persegue por apenas 3 segundos
			if (Date.now() - this.chaseStartTime < 3000) {
				return this.directPursuit(px, py);
			} else {
				// Volta para patrulha após tempo limite
				this.chaseStartTime = null;
			}
		} else {
			this.chaseStartTime = null;
		}

		// Movimento de patrulha mais lento e metódico
		if (distToPoint < 40) {
			this.currentPatrolIndex = (this.currentPatrolIndex + 1) % this.patrolPoints.length;
		}

		const angle = Math.atan2(currentPoint.y - py, currentPoint.x - px);
		// Movimento 30% mais lento na patrulha
		return { dx: Math.cos(angle) * 0.7, dy: Math.sin(angle) * 0.7 };
	},

	randomPursuit(px, py) {
		// Busca aleatória inteligente: explora áreas não visitadas
		if (this.targetDetected) {
			// Persegue apenas por curto período, depois volta à busca
			if (!this.searchChaseStart) {
				this.searchChaseStart = Date.now();
			}
			if (Date.now() - this.searchChaseStart < 2000) {
				return this.directPursuit(px, py);
			} else {
				this.searchChaseStart = null;
				// Continua busca em direção diferente
				this.exploreNewArea(px, py);
			}
		} else {
			this.searchChaseStart = null;
		}

		// Busca sistemática com movimento em espiral
		if (!this.spiralAngle) this.spiralAngle = 0;
		this.spiralAngle += 0.1;
		
		// Movimento em espiral com variação aleatória
		const dx = Math.cos(this.spiralAngle) + (Math.random() - 0.5) * 0.3;
		const dy = Math.sin(this.spiralAngle) + (Math.random() - 0.5) * 0.3;
		
		return this.constrainMovement(px, py, dx, dy);
	},

	constrainMovement(px, py, dx, dy) {
		// Evita que o perseguidor saia das bordas
		const margin = 25; // margem das bordas
		const futureX = px + dx * this.pursuerSpeed;
		const futureY = py + dy * this.pursuerSpeed;

		// Inverte direção se vai sair das bordas
		if (futureX < margin || futureX > this.width - margin) {
			dx *= -0.8;
		}
		if (futureY < margin || futureY > this.height - margin) {
			dy *= -0.8;
		}

		return { dx, dy };
	},

	exploreNewArea(px, py) {
		// Direciona para área menos explorada
		const centerX = this.width / 2;
		const centerY = this.height / 2;
		
		// Se está muito perto do centro, vai para uma borda aleatória
		if (distance(px, py, centerX, centerY) < 100) {
			const edges = [
				{ x: 100, y: 100 }, { x: this.width - 100, y: 100 },
				{ x: this.width - 100, y: this.height - 100 }, { x: 100, y: this.height - 100 }
			];
			const target = edges[Math.floor(Math.random() * edges.length)];
			const angle = Math.atan2(target.y - py, target.x - px);
			this.randomDirection = { dx: Math.cos(angle), dy: Math.sin(angle) };
		} else {
			// Vai para o centro se está nas bordas
			const angle = Math.atan2(centerY - py, centerX - px);
			this.randomDirection = { dx: Math.cos(angle), dy: Math.sin(angle) };
		}
	}
	}
}
</script>

<style scoped>
.simulation-container {
	display: flex;
	gap: 20px;
	padding: 20px;
	background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
	min-height: 80vh;
	font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.canvas-area {
	flex-shrink: 0;
	display: flex;
	flex-direction: column;
	align-items: center;
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
	min-height: 250px; 
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

.strategy-stats {
	margin-top: 15px;
	padding-top: 15px;
	border-top: 2px solid #ecf0f1;
}

.strategy-stats h4 {
	margin: 0 0 10px 0;
	color: #3498db;
	font-size: 14px;
	font-weight: 600;
	text-transform: uppercase;
	letter-spacing: 0.5px;
}

.mini-metric {
	display: flex;
	justify-content: space-between;
	margin-bottom: 5px;
	font-size: 12px;
	color: #7f8c8d;
}

.mini-metric span:last-child {
	font-weight: bold;
	color: #2c3e50;
}

.lost-indicator {
	position: static;
	display: inline-block;
	margin-top: 12px;
	background: #e74c3c;
	color: white;
	padding: 8px 14px;
	border-radius: 8px;
	font-weight: bold;
	text-align: center;
	animation: pulse 1s infinite;
	box-shadow: 0 4px 10px rgba(231, 76, 60, 0.25);
	font-size: 14px;
	border: 2px solid #c0392b;
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

.control-select {
	width: 100%;
	margin-top: 8px;
	padding: 8px 12px;
	border: 2px solid #ecf0f1;
	border-radius: 6px;
	background: #ffffff;
	color: #2c3e50;
	font-size: 14px;
	font-weight: 500;
	cursor: pointer;
	outline: none;
	transition: border-color 0.3s ease;
}

.control-select:focus {
	border-color: #e67e22;
	box-shadow: 0 0 0 3px rgba(230, 126, 34, 0.1);
}

.control-select:hover {
	border-color: #d35400;
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
