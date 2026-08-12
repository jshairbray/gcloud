# Google Cloud Platform Master Container & Serverless Deployment Guide

This document serves as the master operational blueprint for building, styling, animating, and containerizing a public web application with independent asset files, dynamic audio generation, and remote container image exporting workflows.

---

## 1. Environment & Global Workspace Setup
Bind your operational terminal session to your specific Google Cloud infrastructure project parameters.

```bash
# Establish immutable project wide parameters
export PROJECT_ID="my-generic-project-504019"
export REGION="us-central1"
export REPO_NAME="dino-repo"
export IMAGE_NAME="dino-site"

# Synchronize active gcloud context targets
gcloud config set project $PROJECT_ID

# Provision and move into your clean local workspace directory
mkdir -p ~/my-website && cd ~/my-website
```

---

## 2. Decoupled Code & Interactive Architecture
Construct a fully modular web framework. This isolates structural code paths (`index.html`), visual design layouts (`styles.css`), and audio synthesis scripting animations (`script.js`).

### Create index.html
```bash
cat << 'EOF' > index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GCP Dino Profile</title>
    <!-- Standalone external design system reference -->
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="card">
        <div class="status-badge">● Container Live Deploy</div>
        
        <!-- Interactive Clickable Graphic Badge -->
        <div class="dino-container" id="dinoContainer" title="Click me to jump and roar!">
            <span class="dino-emoji" id="dinoEmoji">🦖</span>
        </div>
        
        <h1>Tyrannosaurus Rex</h1>
        <p class="description">The "King of the Tyrant Lizards." A massive, bipedal carnivore that dominated North America during the late Cretaceous period with an incredibly powerful bite force.</p>
        
        <!-- Structural Attributes Matrix -->
        <div class="attributes-grid">
            <div class="attribute-item">
                <span class="attr-label">Diet</span>
                <span class="attr-val">Carnivore</span>
            </div>
            <div class="attribute-item">
                <span class="attr-label">Period</span>
                <span class="attr-val">Cretaceous</span>
            </div>
            <div class="attribute-item">
                <span class="attr-label">Height</span>
                <span class="attr-val">12 - 13 Feet</span>
            </div>
            <div class="attribute-item">
                <span class="attr-label">Weight</span>
                <span class="attr-val">15,000 lbs</span>
            </div>
        </div>
    </div>

    <!-- Frontend Interactive Script Injection -->
    <script src="script.js"></script>
</body>
</html>
EOF
```

### Create styles.css
```bash
cat << 'EOF' > styles.css
:root {
    --bg-color: #f4f7f6;
    --card-bg: #ffffff;
    --text-color: #333333;
    --primary-color: #34a853; /* GCP Green */
    --gray-bg: #f8f9fa;
    --border-color: #e0e0e0;
}
body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    background-color: var(--bg-color);
    color: var(--text-color);
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    margin: 0;
    padding: 20px;
    box-sizing: border-box;
}
.card {
    background: var(--card-bg);
    padding: 2.5rem;
    border-radius: 16px;
    box-shadow: 0 4px 25px rgba(0, 0, 0, 0.06);
    text-align: center;
    max-width: 440px;
    width: 100%;
}
.status-badge {
    background-color: rgba(52, 168, 83, 0.12);
    color: var(--primary-color);
    padding: 0.35rem 0.75rem;
    border-radius: 50px;
    font-size: 0.85rem;
    font-weight: bold;
    display: inline-block;
    margin-bottom: 1.5rem;
}
.dino-container {
    background-color: var(--gray-bg);
    border-radius: 50%;
    width: 110px;
    height: 110px;
    margin: 0 auto 1.5rem auto;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 2px solid var(--border-color);
    cursor: pointer;
    transition: transform 0.2s ease, background-color 0.2s ease;
}
.dino-container:hover {
    background-color: #f0f0f0;
    transform: scale(1.05);
}
.dino-emoji {
    display: inline-block;
    font-size: 3.8rem;
    line-height: 1;
    animation: breathe 3s ease-in-out infinite;
    transform-origin: bottom center;
}
@keyframes breathe {
    0%, 100% { transform: scale(1); }
    50% { transform: scale(1.08) scaleY(0.95); }
}
.dino-jump {
    animation: jumpJump 0.6s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards !important;
}
@keyframes jumpJump {
    0% { transform: translateY(0) scale(1); }
    30% { transform: translateY(-35px) scaleY(1.1) scaleX(0.9) rotate(-10deg); }
    50% { transform: translateY(-40px) rotate(15deg); }
    70% { transform: translateY(-10px) scaleY(0.9) scaleX(1.1); }
    100% { transform: translateY(0) scale(1) rotate(0deg); }
}
h1 {
    color: var(--text-color);
    font-size: 2rem;
    margin: 0 0 0.5rem 0;
}
.description {
    font-size: 0.95rem;
    line-height: 1.6;
    color: #555555;
    margin-bottom: 2rem;
}
.attributes-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;
}
.attribute-item {
    background-color: var(--gray-bg);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
}
.attr-label {
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: #777777;
    margin-bottom: 4px;
}
.attr-val {
    font-size: 0.95rem;
    font-weight: bold;
    color: var(--text-color);
}
EOF
```

### Create script.js
```bash
cat << 'EOF' > script.js
document.addEventListener('DOMContentLoaded', () => {
    const dinoContainer = document.getElementById('dinoContainer');
    const dinoEmoji = document.getElementById('dinoEmoji');

    // Programmatic low-frequency acoustic sound synthesizer
    const triggerDinoRoarSound = () => {
        try {
            const AudioContext = window.AudioContext || window.webkitAudioContext;
            if (!AudioContext) return;
            const ctx = new AudioContext();

            // 1. Establish core low-end throat oscillator
            const osc = ctx.createOscillator();
            osc.type = 'sawtooth';
            osc.frequency.setValueAtTime(80, ctx.currentTime);
            osc.frequency.exponentialRampToValueAtTime(35, ctx.currentTime + 0.4);

            // 2. Introduce chest rattle vibration oscillator
            const frictionOsc = ctx.createOscillator();
            frictionOsc.type = 'triangle';
            frictionOsc.frequency.setValueAtTime(140, ctx.currentTime);
            frictionOsc.frequency.linearRampToValueAtTime(60, ctx.currentTime + 0.5);

            // 3. lowpass filter out sharp synthesized digital spikes
            const filter = ctx.createBiquadFilter();
            filter.type = 'lowpass';
            filter.frequency.setValueAtTime(350, ctx.currentTime);
            filter.frequency.exponentialRampToValueAtTime(180, ctx.currentTime + 0.4);

            // 4. Implement dynamic output volume gain mapping
            const gainNode = ctx.createGain();
            gainNode.gain.setValueAtTime(0.4, ctx.currentTime);
            gainNode.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.6);

            // Stitch audio rendering pipeline nodes together
            osc.connect(filter);
            frictionOsc.connect(filter);
            filter.connect(gainNode);
            gainNode.connect(ctx.destination);

            osc.start();
            frictionOsc.start();
            
            osc.stop(ctx.currentTime + 0.6);
            frictionOsc.stop(ctx.currentTime + 0.6);
        } catch (e) {
            console.warn("Acoustic synthesis block overridden by browser gesture constraints:", e);
        }
    };

    dinoContainer.addEventListener('click', () => {
        if (dinoEmoji.classList.contains('dino-jump')) return;

        dinoEmoji.classList.add('dino-jump');
        triggerDinoRoarSound();

        setTimeout(() => {
            dinoEmoji.classList.remove('dino-jump');
        }, 600);
    });
});
EOF
```

---

## 3. Container System Compilation Structure
Formulate your `Dockerfile` instructions targeting an ultra-lightweight Nginx alpine operating instance.

```bash
cat << 'EOF' > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY styles.css /usr/share/nginx/html/styles.css
COPY script.js /usr/share/nginx/html/script.js
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

---

## 4. Local Cloud Shell Sandbox Testing
Validate container runtime functionality locally on network port `8080` before spending cloud credits.

```bash
# Execute local image compiler pass
docker build -t dino-site:local .

# Fire up a background instance container mapping internal port 80 to external 8080
docker run -d -p 8080:80 --name local-dino-test dino-site:local
```
* **To Preview:** Select the **Web Preview** icon (top right corner header of Cloud Shell) -> click `Preview on port 8080`.
* **To Tear Down:** Clear runtime space when your inspection finishes:
  ```bash
  docker stop local-dino-test && docker rm local-dino-test
  ```

---

## 5. Repository Setup & Remote Cloud Building
