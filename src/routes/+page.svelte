<script>
	import { onMount } from 'svelte';
	import { fade } from 'svelte/transition';

	// State management
	let phase = 'balloon'; // 'balloon' | 'burst' | 'cake' | 'candles' | 'complete'
	let countdown = 10;
	let userInteracted = false;
	let candleCount = 0;
	let showTitle = false;
	/** @type {number[]} */
	let candlesLit = [];
	let isBlowing = false;
    
    // Confetti
    /** @type {{id:number, x:number, y:number, color:string, rotation:number, scale:number, speed:number, swing:number, angle?:number, type: 'pop'|'rain'}[]} */
    let confetti = [];
    /** @type {number|null} */
    let confettiFrameId = null;
    /** @type {number|null} */
    let rainIntervalId = null;
	
	// Balloons bouquet configuration
	const balloonColors = ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff', '#ff6bd6', '#9c27b0'];
	const balloons = [
		{ color: '#ff6b6b', angle: -25, length: 300, scale: 1.1, delay: 0 },
		{ color: '#ffd93d', angle: 20, length: 280, scale: 1.1, delay: 0.2 },
		{ color: '#6bcb77', angle: -10, length: 320, scale: 1.2, delay: 0.4 },
		{ color: '#4d96ff', angle: 10, length: 310, scale: 1.2, delay: 0.6 },
		{ color: '#ff6bd6', angle: 0, length: 290, scale: 1.1, delay: 0.8 },
		{ color: '#9c27b0', angle: -15, length: 330, scale: 1.25, delay: 1.0 }
	];
	let micPermissionDenied = false;
	/** @type {AudioContext | null} */
	let audioContext = null;
	/** @type {AudioContext | null} */
	let musicContext = null;
	let musicPlaying = false;
	let autoplayFailed = false;
	/** @type {AnalyserNode | null} */
	let analyser = null;
	/** @type {MediaStreamAudioSourceNode | null} */
	let microphone = null;
	/** @type {MediaStream | null} */
	let micStream = null;

	// Burst particles
	/** @type {{id:number,x:number,y:number,color:string,size:number,dx:number,dy:number}[]} */
	let particles = [];

	/** @type {ReturnType<typeof setInterval> | null} */
	let countdownInterval = null;

	function unlockAudio() {
		if (!musicContext) return;
		if (musicContext.state === 'suspended') {
			return musicContext.resume();
		}
		return Promise.resolve();
	}

	async function initAudioOnClick() {
		userInteracted = true;
		autoplayFailed = false;
		
		// If we are already playing, let's stop and restart (user clicked because they probably couldn't hear it)
		if (musicPlaying) {
			musicPlaying = false;
			if (musicContext) {
				try { await musicContext.close(); } catch (e) {}
				musicContext = null;
			}
		}
		
		const AudioContextClass = window.AudioContext || /** @type {any} */ (window).webkitAudioContext;
		
		if (!musicContext) {
			musicContext = new AudioContextClass();
		}
		
		try {
			await unlockAudio();
			playHappyBirthday();
		} catch (e) {
			console.error('Audio init failed', e);
		}
	}

	function restartAudio() {
		if (musicContext) {
			try {
				musicContext.close();
			} catch (e) { /* ignore */ }
			musicContext = null;
		}
		musicPlaying = false;
		initAudioOnClick();
	}

	onMount(() => {
		// Start countdown automatically
		countdownInterval = setInterval(() => {
			countdown--;
			if (countdown <= 0) {
				if (countdownInterval) clearInterval(countdownInterval);
				triggerBurst();
			}
		}, 1000);

		return () => {
			if (countdownInterval) clearInterval(countdownInterval);
            if (confettiFrameId) cancelAnimationFrame(confettiFrameId);
            if (rainIntervalId) clearInterval(rainIntervalId);
			if (micStream) {
				for (const track of micStream.getTracks()) track.stop();
				micStream = null;
			}
			if (audioContext) {
				audioContext.close();
			}
			if (musicContext) {
				musicContext.close();
			}
		};
	});

	function startGame() {
		if (phase === 'waiting') {
			phase = 'balloon';
			initAudioOnClick();
			
			countdownInterval = setInterval(() => {
				countdown--;
				if (countdown <= 0) {
					if (countdownInterval) clearInterval(countdownInterval);
					triggerBurst();
				}
			}, 1000);
		}
	}

	function triggerBurst() {
		phase = 'burst';
		// Create burst particles
		particles = Array.from({ length: 30 }, (_, i) => {
			const angle = Math.random() * Math.PI * 2;
			const distance = (Math.random() * 40 + 20) * 6; // px-ish
			return {
			id: i,
			x: 50 + (Math.random() - 0.5) * 20,
			y: 40 + (Math.random() - 0.5) * 20,
			color: ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff', '#ff6bd6'][Math.floor(Math.random() * 5)],
			size: Math.random() * 15 + 5,
			dx: Math.cos(angle) * distance,
			dy: Math.sin(angle) * distance + distance * 0.25
			};
		});

		// After burst animation, show cake
		setTimeout(() => {
			phase = 'cake';
			showTitle = true;
			playHappyBirthday();
			setTimeout(startCandles, 800);
		}, 1200);
	}

    function updateConfetti() {
        if (confetti.length === 0) {
            confettiFrameId = null;
            return;
        }

        confetti = confetti.map(p => {
            if (p.type === 'pop') {
                return {
                    ...p,
                    x: p.x + Math.cos(p.angle ?? 0) * (p.speed * 0.05),
                    y: p.y + Math.sin(p.angle ?? 0) * (p.speed * 0.05) + 0.5, // Gravity
                    rotation: p.rotation + p.speed,
                    speed: p.speed * 0.95 // Friction
                };
            } else {
                // Rain
                return {
                    ...p,
                    y: p.y + p.speed * 0.2,
                    x: p.x + Math.sin(p.y * 0.1) * 0.5,
                    rotation: p.rotation + 2
                };
            }
        }).filter(p => p.y < 110);

        confettiFrameId = requestAnimationFrame(updateConfetti);
    }

    function triggerConfettiPop() {
        // Massive celebration confetti
        const colors = ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff', '#ff6bd6'];
        const newConfetti = Array.from({ length: 400 }, (_, i) => ({
            id: Date.now() + i,
            x: Math.random() * 100, // Anywhere horizontally
            y: Math.random() * 100 - 20, // Anywhere vertically (mostly above fold)
            color: colors[Math.floor(Math.random() * colors.length)],
            rotation: Math.random() * 360,
            scale: Math.random() * 1.5 + 0.5, // Varied sizes
            speed: Math.random() * 10 + 5,
            angle: Math.PI / 2, // Falling down
            swing: Math.random() * 0.2,
            type: /** @type {'pop'} */ ('pop')
        }));
        
        const wasEmpty = confetti.length === 0;
        confetti = [...confetti, ...newConfetti];
        
        if (wasEmpty) {
            updateConfetti();
        }
    }

    function startConfettiRain() {
        if (rainIntervalId) return; // Already raining
        const colors = ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff', '#ff6bd6'];
        
        rainIntervalId = setInterval(() => {
            if (confetti.length > 200) return; // Limit particles
            
            const newPiece = {
                id: Date.now() + Math.random(),
                x: Math.random() * 100,
                y: -10,
                color: colors[Math.floor(Math.random() * colors.length)],
                rotation: Math.random() * 360,
                scale: Math.random() * 1.0 + 0.5,
                speed: Math.random() * 2 + 3,
                swing: Math.random() * 2 - 1,
                type: /** @type {'rain'} */ ('rain')
            };
            
            const wasEmpty = confetti.length === 0;
            confetti = [...confetti, newPiece];
            
            if (wasEmpty) {
                updateConfetti();
            }
        }, 50);
    }

	function startCandles() {
		phase = 'candles';
		candlesLit = [];
		const candleInterval = setInterval(() => {
			candleCount++;
			candlesLit = [...candlesLit, candleCount];
			if (candleCount >= 29) {
				clearInterval(candleInterval);
				setTimeout(() => {
					phase = 'complete';
					setupMicrophone();
				}, 2000);
			}
		}, 150); // Slightly slower to enjoy the progression
	}

	async function setupMicrophone() {
		try {
			audioContext = new AudioContext();
			const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
			micStream = stream;
			microphone = audioContext.createMediaStreamSource(stream);
			analyser = audioContext.createAnalyser();
			analyser.fftSize = 256;
			microphone.connect(analyser);

			const bufferLength = analyser.frequencyBinCount;
			const dataArray = new Uint8Array(bufferLength);

			function detectBlow() {
				if (!analyser) return;
				analyser.getByteFrequencyData(dataArray);
				const average = dataArray.reduce((a, b) => a + b) / bufferLength;
				
				if (average > 50) {
					isBlowing = true;
					blowOutCandles();
				} else {
					isBlowing = false;
				}
				requestAnimationFrame(detectBlow);
			}
			detectBlow();
		} catch (err) {
			console.log('Microphone access denied:', err);
			micPermissionDenied = true;
		}
	}

	function blowOutCandles() {
		if (candlesLit.length > 0) {
			// Blow out 1-3 candles at a time
			const toRemove = Math.min(Math.floor(Math.random() * 3) + 1, candlesLit.length);
			candlesLit = candlesLit.slice(0, -toRemove);
            
            if (candlesLit.length === 0) {
                triggerConfettiPop();
                startConfettiRain();
            }
		}
	}

	function manualBlowOut() {
		if (candlesLit.length > 0) {
			candlesLit = candlesLit.slice(0, -1);
            if (candlesLit.length === 0) {
                triggerConfettiPop();
                startConfettiRain();
            }
		}
	}

	async function playHappyBirthday() {
		if (musicPlaying || !musicContext) return;
		
		// Lock immediately to prevent race conditions
		musicPlaying = true;
		
		const ctx = musicContext;
		
		// Resume context if suspended (browser autoplay policy)
		if (ctx.state === 'suspended') {
			try {
				await ctx.resume();
			} catch (e) {
				console.error('Audio resume failed', e);
				musicPlaying = false; // Reset lock on failure
				return;
			}
		}

		// Double check state after await
		if (ctx.state !== 'running') {
			console.warn('AudioContext still not running');
			musicPlaying = false;
			return;
		}
		
		// Happy Birthday melody notes (frequency in Hz) and durations
		// Note: C4=262, D4=294, E4=330, F4=349, G4=392, A4=440, B4=494, C5=523
		const C4 = 262, D4 = 294, E4 = 330, F4 = 349, G4 = 392, A4 = 440, Bb4 = 466, C5 = 523;
		
		// Happy Birthday melody with timing
		const melody = [
			// "Happy birthday to you"
			{ note: C4, duration: 0.4 },
			{ note: C4, duration: 0.3 },
			{ note: D4, duration: 0.6 },
			{ note: C4, duration: 0.6 },
			{ note: F4, duration: 0.6 },
			{ note: E4, duration: 1.0 },
			// "Happy birthday to you"
			{ note: C4, duration: 0.4 },
			{ note: C4, duration: 0.3 },
			{ note: D4, duration: 0.6 },
			{ note: C4, duration: 0.6 },
			{ note: G4, duration: 0.6 },
			{ note: F4, duration: 1.0 },
			// "Happy birthday dear..."
			{ note: C4, duration: 0.4 },
			{ note: C4, duration: 0.3 },
			{ note: C5, duration: 0.6 },
			{ note: A4, duration: 0.6 },
			{ note: F4, duration: 0.6 },
			{ note: E4, duration: 0.6 },
			{ note: D4, duration: 1.0 },
			// "Happy birthday to you"
			{ note: Bb4, duration: 0.4 },
			{ note: Bb4, duration: 0.3 },
			{ note: A4, duration: 0.6 },
			{ note: F4, duration: 0.6 },
			{ note: G4, duration: 0.6 },
			{ note: F4, duration: 1.2 }
		];
		
		let time = ctx.currentTime + 0.1;
		
		melody.forEach(({ note, duration }) => {
			const oscillator = ctx.createOscillator();
			const gainNode = ctx.createGain();
			
			oscillator.type = 'sine'; // Softer, more pleasant tone
			oscillator.frequency.value = note;
			
			// Gentle envelope
			gainNode.gain.setValueAtTime(0, time);
			gainNode.gain.linearRampToValueAtTime(0.3, time + 0.05);
			gainNode.gain.setValueAtTime(0.3, time + duration - 0.05);
			gainNode.gain.linearRampToValueAtTime(0, time + duration);
			
			oscillator.connect(gainNode);
			gainNode.connect(ctx.destination);
			
			oscillator.start(time);
			oscillator.stop(time + duration);
			
			time += duration;
		});
		
		// Reset playing state after song finishes
		setTimeout(() => {
			musicPlaying = false;
		}, (time - ctx.currentTime) * 1000 + 500);
	}
</script>

<svelte:window 
	on:click={initAudioOnClick} 
	on:keydown={initAudioOnClick}
	on:touchstart={initAudioOnClick}
	on:pointerdown={initAudioOnClick}
/>

<svelte:head>
	<title>Happy Birthday! 🎂</title>
</svelte:head>

<div class="container">

	<!-- Balloon Phase (also serves as start screen) -->
	{#if phase === 'balloon'}
		<div class="balloon-container">
			<div class="balloon">
				<div class="balloon-body">
					<span class="countdown">{countdown}</span>
					<div class="shine"></div>
				</div>
				<div class="balloon-knot"></div>
				<div class="balloon-string"></div>
			</div>
		</div>
	{/if}

	<!-- Burst Phase -->
	{#if phase === 'burst'}
		<div class="burst-container">
			{#each particles as particle}
				<div
					class="particle"
					style="
						--x: {particle.x}%;
						--y: {particle.y}%;
						--color: {particle.color};
						--size: {particle.size}px;
						--dx: {particle.dx}px;
						--dy: {particle.dy}px;
					"
				></div>
			{/each}
		</div>
	{/if}

	<!-- Confetti -->
    {#each confetti as piece (piece.id)}
        <div class="confetti-piece"
            style="
                left: {piece.x}%;
                top: {piece.y}%;
                background: {piece.color};
                transform: rotate({piece.rotation}deg) scale({piece.scale});
            "
        ></div>
    {/each}

	<!-- Cake Phase -->
	{#if phase === 'cake' || phase === 'candles' || phase === 'complete'}
		<div class="cake-section visible">
			<!-- Title (Absolute Positioned) -->
			{#if showTitle}
				<div class="title-container">
					<h1 class="birthday-title">Happy Birthday Abed!</h1>
                    {#if candlesLit.length > 0}
					<p class="wish-text">
                        Make a wish and blow<br>
                        out the candles
                    </p>
                    {/if}
					{#if candlesLit.length === 0 && phase === 'complete'}
						<p class="congrats">Your wish will come true!</p>
					{/if}
				</div>
			{/if}

			<div class="celebration-container">
				<!-- Cake -->
				<div class="cake-container">
					{#if candlesLit.length === 0 && phase === 'complete'}
						<div class="upside-down-face">🙃</div>
					{/if}
					<div class="cake-structure">
						<!-- Top Layer -->
						<div class="layer layer-top">
							<div class="frosting-drip"></div>
                            <div class="frosting-deco"></div>
						</div>
						
						<!-- Middle Layer -->
						<div class="layer layer-middle">
							<div class="frosting-drip"></div>
                            <div class="frosting-deco"></div>
						</div>
						
						<!-- Bottom Layer -->
						<div class="layer layer-bottom">
							<div class="frosting-drip"></div>
                            <div class="frosting-deco"></div>
						</div>
					</div>
                    
                    <!-- Candles Overlay -->
                    <div class="candles-overlay">
                        <!-- Top Candles (5) -->
                        <div class="candles-row top-row">
                            {#each Array(5) as _, i}
                                {#if candleCount >= (24 + i + 1)}
                                    <div class="candle">
                                        {#if candlesLit.includes(24 + i + 1)}
                                        <div class="flame"><div class="flame-inner"></div></div>
                                        {/if}
                                        <div class="candle-stick"></div>
                                    </div>
                                {/if}
                            {/each}
                        </div>
                        
                        <!-- Middle Candles (12) -->
                        <div class="candles-row middle-row">
                            {#each Array(12) as _, i}
                                {#if candleCount >= (12 + i + 1)}
                                    <div class="candle">
                                        {#if candlesLit.includes(12 + i + 1)}
                                        <div class="flame"><div class="flame-inner"></div></div>
                                        {/if}
                                        <div class="candle-stick"></div>
                                    </div>
                                {/if}
                            {/each}
                        </div>
                        
                        <!-- Bottom Candles (12) -->
                        <div class="candles-row bottom-row">
                            {#each Array(12) as _, i}
                                {#if candleCount >= (i + 1)}
                                    <div class="candle">
                                        {#if candlesLit.includes(i + 1)}
                                        <div class="flame"><div class="flame-inner"></div></div>
                                        {/if}
                                        <div class="candle-stick"></div>
                                    </div>
                                {/if}
                            {/each}
                        </div>
                    </div>

					<div class="cake-plate"></div>
				</div>

				<!-- Balloon Bundle -->
				<div class="balloon-bundle">
					{#each balloons as balloon}
						<div class="bundle-item" style="--color: {balloon.color}; --delay: {balloon.delay}s; --angle: {balloon.angle}deg; --length: {balloon.length}px; --scale: {balloon.scale}">
							<div class="balloon-shape">
								<div class="shine"></div>
							</div>
                            <!-- Wavy String -->
                            <svg class="string-svg" style="height: {balloon.length}px" viewBox="0 0 20 100" preserveAspectRatio="none">
                                <path class="string-path" d="M10,0 C20,33 0,66 10,100" vector-effect="non-scaling-stroke" />
                            </svg>
						</div>
					{/each}
                    <div class="knot"></div>
				</div>
			</div>

			<!-- Manual blow button if mic denied -->
			{#if phase === 'complete' && candlesLit.length > 0}
				<div class="blow-section">
					{#if micPermissionDenied}
						<button class="blow-button" on:click={manualBlowOut}>
							💨 Click to blow out candles
						</button>
					{:else}
						<p class="blow-hint" class:active={isBlowing}>
							{isBlowing ? '💨 Blowing...' : '🎤 Blow into your microphone!'}
						</p>
					{/if}
					<p class="candle-count">{candlesLit.length} candles remaining</p>
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}

	.container {
		min-height: 100vh;
		background: linear-gradient(135deg, #fff9f0 0%, #ffffff 50%, #fff5eb 100%);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		font-family: 'Segoe UI', system-ui, sans-serif;
		position: relative;
		overflow: hidden;
	}

	/* Balloon Styles */
	.balloon-container {
		animation: float 3s ease-in-out infinite;
		background: none;
		border: none;
		cursor: pointer;
		padding: 0;
	}

	@keyframes float {
		0%, 100% { transform: translateY(0); }
		50% { transform: translateY(-20px); }
	}

	.balloon {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.balloon-body {
		width: 180px;
		height: 220px;
		background: linear-gradient(135deg, #ff6b6b 0%, #ee5a5a 50%, #d63031 100%);
		border-radius: 50% 50% 50% 50% / 40% 40% 60% 60%;
		position: relative;
		box-shadow: 
			inset -20px -20px 40px rgba(0,0,0,0.2),
			inset 20px 20px 40px rgba(255,255,255,0.2),
			0 20px 60px rgba(0,0,0,0.3);
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.shine {
		position: absolute;
		top: 30px;
		left: 30px;
		width: 40px;
		height: 60px;
		background: rgba(255,255,255,0.3);
		border-radius: 50%;
		transform: rotate(-30deg);
	}

	.countdown {
		font-size: 80px;
		font-weight: bold;
		color: white;
		text-shadow: 2px 2px 10px rgba(0,0,0,0.3);
		z-index: 1;
	}

	.balloon-knot {
		width: 20px;
		height: 20px;
		background: #d63031;
		clip-path: polygon(50% 0%, 0% 100%, 100% 100%);
		margin-top: -5px;
	}

	.balloon-string {
		width: 2px;
		height: 100px;
		background: linear-gradient(to bottom, #d63031, #888);
		animation: sway 2s ease-in-out infinite;
	}

	@keyframes sway {
		0%, 100% { transform: rotate(-5deg); }
		50% { transform: rotate(5deg); }
	}

	/* Burst Particles */
	.burst-container {
		position: absolute;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	.particle {
		position: absolute;
		left: var(--x);
		top: var(--y);
		width: var(--size);
		height: var(--size);
		background: var(--color);
		border-radius: 50%;
		animation: explode 1.2s ease-out forwards;
	}

	@keyframes explode {
		0% {
			transform: translate(0, 0) scale(1);
			opacity: 1;
		}
		100% {
			transform: translate(var(--dx), var(--dy)) scale(0);
			opacity: 0;
		}
	}

	/* Cake Section */
	.cake-section {
		display: flex;
		flex-direction: column;
		align-items: center;
		opacity: 0;
		transform: scale(0.8);
		transition: all 0.8s cubic-bezier(0.34, 1.56, 0.64, 1);
	}

	.cake-section.visible {
		opacity: 1;
		transform: scale(1);
	}

	/* Cake Phase Container */
	.celebration-container {
		display: flex;
		align-items: flex-end;
		justify-content: center;
		gap: 60px;
		margin-top: 0px;
	}

	/* Title */
	.title-container {
		text-align: center;
		margin-bottom: 0px;
		animation: fadeInDown 0.8s ease-out;
		position: relative;
		width: 100%;
		z-index: 20;
	}

	.birthday-title {
		font-family: 'Georgia', 'Times New Roman', serif;
		font-size: clamp(2rem, 6vw, 3.5rem);
		color: #ffd93d;
		text-shadow: 
			2px 2px 0px #d4a017,
			4px 4px 10px rgba(0,0,0,0.2);
		margin: 50px 0 15px 0;
		letter-spacing: 2px;
		animation: floatTitle 3s ease-in-out infinite;
        white-space: nowrap;
	}

	.wish-text {
		font-family: 'Georgia', 'Times New Roman', serif;
		font-size: 1rem;
		color: #888;
		margin-top: 10px;
		animation: floatTitle 3s ease-in-out infinite;
		animation-delay: 0.5s;
        text-transform: uppercase;
        letter-spacing: 2px;
	}

    .confetti-piece {
        border-radius: 50%;
        position: absolute;
        width: 10px;
        height: 10px;
        pointer-events: none;
        z-index: 50;
    }

	@keyframes floatTitle {
		0%, 100% { transform: translateY(0); }
		50% { transform: translateY(-10px); }
	}

	/* Realistic Cake */
	.cake-container {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
		margin-top: 20px;
	}

	.cake-structure {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.layer {
		background: linear-gradient(to right, #f8b195, #f67280);
		border-radius: 5px;
		position: relative;
		box-shadow: inset 0 0 20px rgba(0,0,0,0.1);
		display: flex;
		justify-content: center;
	}

	.layer-top {
		width: 220px;
		height: 60px;
		background: linear-gradient(to right, #ff9a9e, #fad0c4);
		border-radius: 10px 10px 5px 5px;
		z-index: 3;
	}

	.layer-middle {
		width: 280px;
		height: 70px;
		background: linear-gradient(to right, #ff9a9e, #fbc2eb);
		border-radius: 5px;
		z-index: 2;
		margin-top: -5px; /* Overlap */
	}

	.layer-bottom {
		width: 340px;
		height: 80px;
		background: linear-gradient(to right, #ff9a9e, #fbc2eb);
		border-radius: 5px 5px 15px 15px;
		z-index: 1;
		margin-top: -5px; /* Overlap */
	}

	.frosting-drip {
		position: absolute;
		top: -10px;
		width: 105%;
		height: 25px;
		background: #fff;
		border-radius: 15px;
		left: -2.5%;
        box-shadow: 0 2px 5px rgba(0,0,0,0.1);
	}
    
    .frosting-deco {
        position: absolute;
        top: 50%;
        left: 0;
        width: 100%;
        height: 10px;
        background-image: radial-gradient(circle, rgba(255,255,255,0.5) 2px, transparent 2.5px);
        background-size: 15px 15px;
        opacity: 0.6;
    }

	.cake-plate {
		width: 380px;
		height: 15px;
		background: #eee;
		border-radius: 50%;
		margin-top: 5px;
		box-shadow: 0 10px 20px rgba(0,0,0,0.1);
	}

	/* Candles Overlay */
    .candles-overlay {
        position: absolute;
        bottom: 25px; /* Above plate */
        width: 100%;
        height: 220px;
        pointer-events: none;
        z-index: 10;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

	.candles-row {
		display: flex;
		justify-content: center;
        position: absolute;
        width: 100%;
	}

	.top-row {
		gap: 20px;
		bottom: 180px;
        z-index: 13;
	}

	.middle-row {
		gap: 12px;
		bottom: 125px;
        z-index: 12;
	}

	.bottom-row {
		gap: 14px;
		bottom: 60px;
        z-index: 11;
	}

	.candle {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
		animation: candleAppear 0.3s ease-out backwards;
	}

	.candle-stick {
		width: 6px;
		height: 40px;
		background: repeating-linear-gradient(
			45deg,
			#fff,
			#fff 4px,
			#ff6b9d 4px,
			#ff6b9d 8px
		);
		border-radius: 3px;
		box-shadow: 1px 1px 2px rgba(0,0,0,0.2);
	}

	/* Balloon Bouquet */
	.balloon-bundle {
		position: relative;
		width: 100px;
		height: 400px;
        /* Aligns with flex-end of container (the cake plate) */
        margin-left: 20px;
	}

	.bundle-item {
		position: absolute;
		bottom: 0;
		left: 50%;
        width: 80px;
        height: var(--length);
		transform-origin: bottom center;
		transform: translateX(-50%) rotate(var(--angle)) scale(var(--scale));
		animation: swayBundle 5s ease-in-out infinite alternate;
		animation-delay: var(--delay);
		display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: flex-start;
	}

    @keyframes swayBundle {
        from { transform: translateX(-50%) rotate(calc(var(--angle) - 3deg)) scale(var(--scale)); }
        to { transform: translateX(-50%) rotate(calc(var(--angle) + 3deg)) scale(var(--scale)); }
    }
    
    .string-svg {
        width: 20px;
        /* height set by inline style */
        overflow: visible;
        margin-top: -5px;
        z-index: 1;
        opacity: 0.6;
    }
    
    .string-path {
        fill: none;
        stroke: #555;
        stroke-width: 1.5px;
    }
    
    .knot {
        position: absolute;
        width: 8px;
        height: 8px;
        background: #555;
        border-radius: 50%;
        bottom: -4px;
        left: 50%;
        transform: translateX(-50%);
        z-index: 10;
        box-shadow: 0 0 2px rgba(0,0,0,0.3);
    }

	.balloon-shape {
		width: 80px;
		height: 96px;
		background: var(--color);
		border-radius: 50% 50% 50% 50% / 40% 40% 60% 60%;
		position: relative;
		box-shadow: 
			inset -10px -10px 20px rgba(0,0,0,0.1),
			inset 10px 10px 20px rgba(255,255,255,0.2),
			2px 2px 10px rgba(0,0,0,0.1);
        z-index: 2;
        animation: floatLocal 4s ease-in-out infinite;
	}

    @keyframes floatLocal {
        0%, 100% { transform: translateY(0); }
        50% { transform: translateY(-5px); }
    }

	.shine {
		position: absolute;
		top: 15px;
		left: 15px;
		width: 20px;
		height: 35px;
		background: rgba(255,255,255,0.4);
		border-radius: 50%;
		transform: rotate(-30deg);
	}

	.flame {
		width: 18px;
		height: 28px;
		background: linear-gradient(to top, #ff9500, #ffcc00, #fff);
		border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%;
		animation: flicker 0.3s ease-in-out infinite alternate;
		position: relative;
		margin-bottom: -4px;
	}

	.flame-inner {
		position: absolute;
		bottom: 3px;
		left: 50%;
		transform: translateX(-50%);
		width: 8px;
		height: 12px;
		background: linear-gradient(to top, #4d96ff, #fff);
		border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%;
	}

	@keyframes flicker {
		0% { transform: scale(1) rotate(-3deg); }
		100% { transform: scale(1.15) rotate(3deg); }
	}

	/* Blow Section */
	.blow-section {
		margin-top: 30px;
		text-align: center;
	}

	.blow-button {
		padding: 15px 30px;
		font-size: 1.2rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 10px 30px rgba(102, 126, 234, 0.4);
	}

	.blow-button:hover {
		transform: translateY(-3px);
		box-shadow: 0 15px 40px rgba(102, 126, 234, 0.5);
	}

	.blow-button:active {
		transform: translateY(0);
	}

	.blow-hint {
		font-size: 1.2rem;
		color: #666;
		padding: 15px 30px;
		background: rgba(255,255,255,0.1);
		border-radius: 50px;
		transition: all 0.3s ease;
	}

	.blow-hint.active {
		background: rgba(107, 203, 119, 0.3);
		color: #6bcb77;
	}

	.candle-count {
		color: rgba(0,0,0,0.5);
		margin-top: 10px;
		font-size: 1rem;
	}

	.congrats {
		font-family: 'Georgia', 'Times New Roman', serif;
		font-size: clamp(1.5rem, 5vw, 2.5rem);
		color: #6bcb77;
		text-shadow: 
			2px 2px 0px #d4a017,
			4px 4px 10px rgba(0,0,0,0.2);
		margin-top: 10px;
		letter-spacing: 2px;
		animation: floatTitle 3s ease-in-out infinite;
		white-space: nowrap;
	}
	.upside-down-face {
		font-size: 4rem;
		position: absolute;
		top: -70px;
		animation: float 3s ease-in-out infinite;
		z-index: 20;
	}

</style>
