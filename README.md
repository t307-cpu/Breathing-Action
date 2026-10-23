<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mechanism of Breathing Simulator</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f8fafc;
        }
        .active-step {
            background-color: #fef2f2;
            border-left: 4px solid #ef4444;
        }
        .active-step-exhale {
            background-color: #eff6ff;
            border-left: 4px solid #3b82f6;
        }
        .inactive-step {
            opacity: 0.5;
            border-left: 4px solid transparent;
        }
        svg {
            drop-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
    </style>
</head>
<body class="p-4 sm:p-8 flex justify-center items-center min-h-screen">

    <div class="max-w-6xl w-full bg-white rounded-2xl shadow-2xl overflow-hidden flex flex-col lg:flex-row border border-gray-200">
        
        <!-- Left Visual Panel -->
        <div class="w-full lg:w-5/12 bg-slate-100 p-6 flex flex-col items-center justify-center border-r border-gray-200 relative min-h-[500px]">
            
            <!-- Mode Toggle -->
            <div class="absolute top-6 left-0 right-0 flex justify-center gap-2 z-10 px-4">
                <button id="tab-inhale" class="px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-red-500 text-white">Inhalation</button>
                <button id="tab-exhale" class="px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-white text-gray-600 hover:bg-gray-50">Exhalation</button>
            </div>

            <div class="relative w-full max-w-sm mt-12 aspect-[3/4] bg-white rounded-2xl shadow-inner border border-gray-200 overflow-hidden flex justify-center items-center">
                <svg viewBox="0 0 400 500" class="w-full h-full" id="lung-svg">
                    <!-- Head & Torso Silhouette -->
                    <path d="M 120 50 C 120 20, 280 20, 280 50 C 280 90, 310 110, 330 150 C 360 220, 380 350, 380 500 L 20 500 C 20 350, 40 220, 70 150 C 90 110, 120 90, 120 50 Z" fill="#fdf2f8" opacity="0.8" />
                    
                    <!-- Trachea -->
                    <path d="M 185 80 L 185 170 M 215 80 L 215 170" stroke="#cbd5e1" stroke-width="8" stroke-linecap="round" fill="none" />

                    <!-- Dynamic Elements (Paths updated via JS) -->
                    <!-- Rib Cage (Thoracic Cavity) Background -->
                    <path id="cavity-bg" fill="#f1f5f9" stroke="#94a3b8" stroke-width="6" stroke-dasharray="8 6" stroke-linecap="round" />
                    
                    <!-- Lungs -->
                    <path id="lungs" fill="#fca5a5" stroke="#ef4444" stroke-width="3" opacity="0.95" />

                    <!-- Diaphragm -->
                    <path id="diaphragm" fill="none" stroke="#ea580c" stroke-width="12" stroke-linecap="round" />
                    <text id="dia-label" x="200" y="430" fill="#ea580c" font-size="16" font-weight="extrabold" text-anchor="middle" stroke="white" stroke-width="4" stroke-linejoin="round" paint-order="stroke">Diaphragm</text>

                    <!-- Airflow Arrows -->
                    <g id="arrows-in" opacity="0">
                        <path d="M 200 20 L 200 90" stroke="#ef4444" stroke-width="6" fill="none" />
                        <polygon points="185,75 215,75 200,105" fill="#ef4444" />
                        <text x="230" y="60" fill="#ef4444" font-weight="bold">Air IN</text>
                    </g>
                    <g id="arrows-out" opacity="0">
                        <path d="M 200 105 L 200 35" stroke="#3b82f6" stroke-width="6" fill="none" />
                        <polygon points="185,50 215,50 200,20" fill="#3b82f6" />
                        <text x="230" y="60" fill="#3b82f6" font-weight="bold">Air OUT</text>
                    </g>

                    <!-- Pressure Indicator (Appears Step 4) -->
                    <g id="pressure-indicator" opacity="0" transform="translate(200, 250)">
                        <circle cx="0" cy="0" r="30" fill="white" opacity="0.8" />
                        <text id="pressure-text" x="0" y="5" text-anchor="middle" font-weight="bold" font-size="24">-</text>
                    </g>
                </svg>
            </div>
            
            <!-- Controls -->
            <div class="mt-6 flex gap-4 w-full max-w-sm justify-between">
                <button id="btn-prev" class="px-5 py-3 bg-white border-2 border-gray-300 text-gray-700 font-bold rounded-xl hover:bg-gray-50 disabled:opacity-40 disabled:cursor-not-allowed transition w-1/2">
                    &larr; Previous
                </button>
                <button id="btn-next" class="px-5 py-3 bg-slate-800 text-white font-bold rounded-xl hover:bg-slate-700 disabled:opacity-40 disabled:cursor-not-allowed transition w-1/2 shadow-lg">
                    Next Step &rarr;
                </button>
            </div>
        </div>

        <!-- Right Text Panel -->
        <div class="w-full lg:w-7/12 p-8 flex flex-col bg-white">
            <h1 class="text-3xl font-extrabold text-slate-800 mb-2">Mechanism of Breathing</h1>
            <p class="text-gray-500 mb-6 font-medium">Follow the exact sequence of biological events.</p>
            
            <div id="steps-container" class="flex-1 space-y-3 overflow-y-auto pr-2">
                <!-- Steps injected by JS -->
            </div>
        </div>
    </div>

    <script>
        const sequences = {
            inhalation: [
                {
                    text: "<strong>Initial State:</strong> The respiratory system is at rest.",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 }
                },
                {
                    text: "<strong>Step 1:</strong> Intercostal muscles <span class='text-red-600 font-bold'>contract</span> (rib cage moves <span class='text-red-600 font-bold'>outwards</span> and <span class='text-red-600 font-bold'>upwards</span>) AND diaphragm muscles <span class='text-red-600 font-bold'>contract</span> (diaphragm <span class='text-red-600 font-bold'>flattened</span>).",
                    targets: { cav: 1, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 } // Cavity expands, gap forms
                },
                {
                    text: "<strong>Step 2:</strong> Thoracic cavity volume <span class='text-red-600 font-bold'>increases</span>.",
                    targets: { cav: 1, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 }
                },
                {
                    text: "<strong>Step 3:</strong> Lungs expand (Lung volume <span class='text-red-600 font-bold'>increases</span> to perfectly fill the new space).",
                    targets: { cav: 1, lung: 1, airIn: 0, airOut: 0, pressureOp: 0 } // Lungs expand to cavity
                },
                {
                    text: "<strong>Step 4:</strong> Lung pressure <span class='text-red-600 font-bold'>decreases</span> (becomes lower than atmospheric pressure).",
                    targets: { cav: 1, lung: 1, airIn: 0, airOut: 0, pressureOp: 1, pressureSign: "-" }
                },
                {
                    text: "<strong>Step 5:</strong> Air <span class='text-red-600 font-bold'>flows into</span> the lungs.",
                    targets: { cav: 1, lung: 1, airIn: 1, airOut: 0, pressureOp: 1, pressureSign: "-" }
                }
            ],
            exhalation: [
                {
                    text: "<strong>Initial State:</strong> Lungs are full (end of inhalation).",
                    targets: { cav: 1, lung: 1, airIn: 0, airOut: 0, pressureOp: 0 }
                },
                {
                    text: "<strong>Step 1:</strong> Intercostal muscles <span class='text-blue-600 font-bold'>relax</span> (rib cage moves <span class='text-blue-600 font-bold'>inwards</span> and <span class='text-blue-600 font-bold'>downwards</span>) AND diaphragm muscles <span class='text-blue-600 font-bold'>relax</span> (diaphragm recoils to <span class='text-blue-600 font-bold'>dome shape</span>).",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 } // Cavity shrinks, forces lungs to shrink simultaneously to prevent overlap
                },
                {
                    text: "<strong>Step 2:</strong> Thoracic cavity volume <span class='text-blue-600 font-bold'>decreases</span>.",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 }
                },
                {
                    text: "<strong>Step 3:</strong> Lungs shrink back (Lung volume <span class='text-blue-600 font-bold'>decreases</span>).",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 }
                },
                {
                    text: "<strong>Step 4:</strong> Lung pressure <span class='text-blue-600 font-bold'>increases</span> (becomes higher than atmospheric pressure).",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 1, pressureSign: "+" }
                },
                {
                    text: "<strong>Step 5:</strong> Air <span class='text-blue-600 font-bold'>flows out</span> of the lungs.",
                    targets: { cav: 0, lung: 0, airIn: 0, airOut: 1, pressureOp: 1, pressureSign: "+" }
                }
            ]
        };

        let currentMode = 'inhalation';
        let currentStep = 0;

        // Visual state variables (0 to 1) for smooth interpolation
        let state = { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 };
        let targetState = { cav: 0, lung: 0, airIn: 0, airOut: 0, pressureOp: 0 };

        // DOM Elements
        const stepsContainer = document.getElementById('steps-container');
        const btnPrev = document.getElementById('btn-prev');
        const btnNext = document.getElementById('btn-next');
        const tabInhale = document.getElementById('tab-inhale');
        const tabExhale = document.getElementById('tab-exhale');
        
        // SVG Elements
        const pathCavity = document.getElementById('cavity-bg');
        const pathLungs = document.getElementById('lungs');
        const pathDiaphragm = document.getElementById('diaphragm');
        const diaLabel = document.getElementById('dia-label');
        const arrowsIn = document.getElementById('arrows-in');
        const arrowsOut = document.getElementById('arrows-out');
        const pressureIndicator = document.getElementById('pressure-indicator');
        const pressureText = document.getElementById('pressure-text');

        function initMode(mode) {
            currentMode = mode;
            currentStep = 0;
            
            // Instantly snap the visuals to the start of the chosen mode
            let initialTarget = sequences[mode][0].targets;
            state = { ...initialTarget }; 
            targetState = { ...initialTarget };

            // Update Tabs
            if (mode === 'inhalation') {
                tabInhale.className = "px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-red-500 text-white";
                tabExhale.className = "px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-white text-gray-600 hover:bg-gray-50";
            } else {
                tabExhale.className = "px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-blue-500 text-white";
                tabInhale.className = "px-6 py-2 rounded-full font-bold text-sm shadow transition-colors bg-white text-gray-600 hover:bg-gray-50";
            }

            renderStepList();
            updateUI();
        }

        function renderStepList() {
            stepsContainer.innerHTML = '';
            sequences[currentMode].forEach((stepData, index) => {
                const div = document.createElement('div');
                div.className = `p-4 rounded-lg transition-all duration-300 ${index === 0 ? 'hidden' : ''}`; // Hide Initial State text
                
                // Styling active vs inactive
                if (index === currentStep) {
                    div.classList.add(currentMode === 'inhalation' ? 'active-step' : 'active-step-exhale');
                } else {
                    div.classList.add('inactive-step', 'bg-gray-50');
                }
                
                div.innerHTML = `<p class="text-lg text-gray-800">${stepData.text}</p>`;
                stepsContainer.appendChild(div);
            });
        }

        function updateUI() {
            renderStepList();
            btnPrev.disabled = currentStep === 0;
            btnNext.disabled = currentStep === sequences[currentMode].length - 1;
            
            const curTargets = sequences[currentMode][currentStep].targets;
            targetState = { ...curTargets };
            
            if (curTargets.pressureSign) {
                pressureText.textContent = curTargets.pressureSign;
                pressureText.setAttribute('fill', curTargets.pressureSign === '-' ? '#ef4444' : '#3b82f6');
            }
        }

        btnPrev.addEventListener('click', () => {
            if (currentStep > 0) { currentStep--; updateUI(); }
        });

        btnNext.addEventListener('click', () => {
            if (currentStep < sequences[currentMode].length - 1) { currentStep++; updateUI(); }
        });

        tabInhale.addEventListener('click', () => initMode('inhalation'));
        tabExhale.addEventListener('click', () => initMode('exhalation'));

        // Core Render Loop for mathematical SVG generation
        function animate() {
            // Lerp towards target state
            const speed = 0.12;
            state.cav += (targetState.cav - state.cav) * speed;
            state.lung += (targetState.lung - state.lung) * speed;
            state.airIn += (targetState.airIn - state.airIn) * speed;
            state.airOut += (targetState.airOut - state.airOut) * speed;
            state.pressureOp += (targetState.pressureOp - state.pressureOp) * speed;

            let c = state.cav;   // 0 (Rest/Dome) to 1 (Expanded/Flat)
            let l = state.lung;  // 0 (Rest size) to 1 (Expanded size)

            // Calculate bounding coordinates
            // Cavity Geometry (Ribs & Diaphragm Base)
            let cavLeft = 120 - (c * 25);
            let cavRight = 280 + (c * 25);
            let topY = 160;
            let diaCenterY = 320 + (c * 70);
            let diaEndY = 360 + (c * 20);

            // Lung Geometry (Must never exceed cavity bounds physically)
            // To ensure they never cross the diaphragm, lung interpolation matches cavity when l=c.
            let lungLeft = 125 - (l * 25);
            let lungRight = 275 + (l * 25);
            let lungDiaCenterY = 315 + (l * 70); 
            let lungDiaEndY = 355 + (l * 20);

            // Generate Paths
            // 1. Diaphragm Path
            let dPath = `M ${cavLeft - 10} ${diaEndY} Q 200 ${diaCenterY} ${cavRight + 10} ${diaEndY}`;
            pathDiaphragm.setAttribute('d', dPath);
            // Lock label horizontally in the center and vertically below the maximum flattened depth (390)
            diaLabel.setAttribute('x', '200');
            diaLabel.setAttribute('y', '430');

            // 2. Thoracic Cavity Path (Ribs + bottom)
            let cavPath = `M ${cavLeft} ${diaEndY} C ${cavLeft} 220, 160 ${topY}, 200 ${topY} C 240 ${topY}, ${cavRight} 220, ${cavRight} ${diaEndY} Q 200 ${diaCenterY} ${cavLeft} ${diaEndY} Z`;
            pathCavity.setAttribute('d', cavPath);

            // 3. Lungs Path (Perfectly fits inside when l == c)
            let lPath = `M ${lungLeft} ${lungDiaEndY} C ${lungLeft} 220, 165 ${topY + 5}, 200 ${topY + 5} C 235 ${topY + 5}, ${lungRight} 220, ${lungRight} ${lungDiaEndY} Q 200 ${lungDiaCenterY} ${lungLeft} ${lungDiaEndY} Z`;
            pathLungs.setAttribute('d', lPath);

            // 4. Opacities & Transforms
            arrowsIn.setAttribute('opacity', state.airIn);
            arrowsOut.setAttribute('opacity', state.airOut);
            arrowsIn.setAttribute('transform', `translate(0, ${-20 + (state.airIn * 20)})`);
            arrowsOut.setAttribute('transform', `translate(0, ${20 - (state.airOut * 20)})`);
            
            pressureIndicator.setAttribute('opacity', state.pressureOp);
            
            // Visual Squeeze Effect for Exhalation Step 3
            if (currentMode === 'exhalation' && currentStep >= 3) {
                pathLungs.setAttribute('fill', '#f87171'); // Darker pink indicating squished
            } else {
                pathLungs.setAttribute('fill', '#fca5a5'); // Normal pink
            }

            requestAnimationFrame(animate);
        }

        // Boot
        initMode('inhalation');
        requestAnimationFrame(animate);

    </script>
</body>
</html>
