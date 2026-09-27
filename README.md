<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Free Automated BBS Generator</title>
    <!-- Tailwind CSS for modern styling -->
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-slate-50 text-slate-800 font-sans">

    <div class="max-w-6xl mx-auto p-6">
        <header class="mb-8 text-center">
            <h1 class="text-3xl font-bold text-blue-600">Automated Bar Bending Schedule (BBS)</h1>
            <p class="text-sm text-slate-500 mt-1">Generate shape codes, dynamic bend-wise drawings, and cutting lengths for free.</p>
        </header>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <!-- Input Panel -->
            <div class="bg-white p-6 rounded-xl shadow-md space-y-4 md:col-span-1">
                <h2 class="text-xl font-semibold border-b pb-2">Bar Parameters</h2>
                
                <div>
                    <label class="block text-sm font-medium text-slate-700">Select Shape Type</label>
                    <select id="shapeType" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" onchange="updateInputs()">
                        <option value="straight">Straight Bar (Code 00)</option>
                        <option value="lshape">L-Shape / Bend (Code 11)</option>
                        <option value="stirrup">Stirrup / Link (Code 21)</option>
                    </select>
                </div>

                <div>
                    <label class="block text-sm font-medium text-slate-700">Bar Diameter (mm - $\phi$)</label>
                    <input type="number" id="dia" value="16" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" oninput="calculateBBS()">
                </div>

                <div id="dimA_container">
                    <label class="block text-sm font-medium text-slate-700" id="labelA">Length A (mm)</label>
                    <input type="number" id="dimA" value="2000" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" oninput="calculateBBS()">
                </div>

                <div id="dimB_container">
                    <label class="block text-sm font-medium text-slate-700" id="labelB">Length B (mm)</label>
                    <input type="number" id="dimB" value="500" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" oninput="calculateBBS()">
                </div>

                <div id="dimC_container" class="hidden">
                    <label class="block text-sm font-medium text-slate-700" id="labelC">Length C (mm)</label>
                    <input type="number" id="dimC" value="300" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" oninput="calculateBBS()">
                </div>

                <div>
                    <label class="block text-sm font-medium text-slate-700">Number of Bars (Nos)</label>
                    <input type="number" id="nos" value="10" class="mt-1 block w-full rounded-md border-slate-300 shadow-sm p-2 border" oninput="calculateBBS()">
                </div>
            </div>

            <!-- Visualization & Results Panel -->
            <div class="bg-white p-6 rounded-xl shadow-md md:col-span-2 flex flex-col justify-between">
                <div>
                    <h2 class="text-xl font-semibold border-b pb-2 mb-4">Dynamic Shape Visualizer</h2>
                    <!-- SVG Canvas for Shape Rendering -->
                    <div class="border-2 border-dashed border-slate-200 rounded-lg p-4 flex items-center justify-center bg-slate-50 h-64">
                        <svg id="shapeSvg" class="w-full h-full" viewBox="0 0 400 200"></svg>
                    </div>
                </div>

                <div class="mt-6 bg-blue-50 p-4 rounded-lg border border-blue-100">
                    <h3 class="font-semibold text-blue-900 mb-2">Calculation Output</h3>
                    <div class="grid grid-cols-2 gap-4 text-sm">
                        <div>
                            <span class="text-slate-600">Cutting Length per Bar:</span>
                            <p id="cutLength" class="font-bold text-lg text-blue-700">0 mm</p>
                        </div>
                        <div>
                            <span class="text-slate-600">Total Steel Weight:</span>
                            <p id="totalWeight" class="font-bold text-lg text-blue-700">0 kg</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        function updateInputs() {
            const type = document.getElementById('shapeType').value;
            const dimBContainer = document.getElementById('dimB_container');
            const dimCContainer = document.getElementById('dimC_container');
            const labelA = document.getElementById('labelA');
            const labelB = document.getElementById('labelB');

            if (type === 'straight') {
                dimBContainer.classList.add('hidden');
                dimCContainer.classList.add('hidden');
                labelA.innerText = "Total Length A (mm)";
            } else if (type === 'lshape') {
                dimBContainer.classList.remove('hidden');
                dimCContainer.classList.add('hidden');
                labelA.innerText = "Leg A (mm)";
                labelB.innerText = "Leg B (mm)";
            } else if (type === 'stirrup') {
                dimBContainer.classList.remove('hidden');
                dimCContainer.classList.remove('hidden');
                labelA.innerText = "Width A (mm)";
                labelB.innerText = "Height B (mm)";
                labelC.innerText = "Hook Length C (mm)";
            }
            calculateBBS();
        }

        function calculateBBS() {
            const type = document.getElementById('shapeType').value;
            const dia = parseFloat(document.getElementById('dia').value) || 0;
            const a = parseFloat(document.getElementById('dimA').value) || 0;
            const b = parseFloat(document.getElementById('dimB').value) || 0;
            const c = parseFloat(document.getElementById('dimC').value) || 0;
            const nos = parseInt(document.getElementById('nos').value) || 0;

            let cuttingLength = 0;
            let svgHtml = '';

            // Standard Bend Deductions (approx standard rules)
            // 90 deg bend = 2*dia, 135 deg bend = 3*dia
            if (type === 'straight') {
                cuttingLength = a;
                svgHtml = `<line x1="50" y1="100" x2="350" y2="100" stroke="#2563eb" stroke-width="6" stroke-linecap="round"/>
                           <text x="200" y="85" text-anchor="middle" font-size="14" fill="#1e293b">A = ${a} mm</text>`;
            } else if (type === 'lshape') {
                // 1 bend of 90 deg -> deduct 2*dia
                let bendDeduction = 2 * dia;
                cuttingLength = (a + b) - bendDeduction;
                svgHtml = `<path d="M 80 150 L 80 70 L 300 70" fill="none" stroke="#2563eb" stroke-width="6" stroke-linecap="round" stroke-linejoin="round"/>
                           <text x="65" y="110" text-anchor="end" font-size="14" fill="#1e293b">A = ${a}</text>
                           <text x="190" y="55" text-anchor="middle" font-size="14" fill="#1e293b">B = ${b}</text>`;
            } else if (type === 'stirrup') {
                // 4 bends of 90 deg (8*dia) + 2 hooks (2 * C)
                let bendDeduction = 8 * dia;
                cuttingLength = (2 * (a + b)) + (2 * c) - bendDeduction;
                svgHtml = `<rect x="120" y="60" width="160" height="100" rx="5" fill="none" stroke="#2563eb" stroke-width="6"/>
                           <text x="200" y="45" text-anchor="middle" font-size="14" fill="#1e293b">Width A = ${a}</text>
                           <text x="95" y="115" text-anchor="middle" font-size="14" fill="#1e293b">B = ${b}</text>`;
            }

            // Weight calculation: (D^2 / 162.2) * (Length in meters) * Nos
            let lengthInMeters = Math.max(0, cuttingLength) / 1000;
            let unitWeight = (Math.pow(dia, 2) / 162.2);
            let totalWeight = unitWeight * lengthInMeters * nos;

            document.getElementById('cutLength').innerText = `${cuttingLength.toFixed(1)} mm (${lengthInMeters.toFixed(3)} m)`;
            document.getElementById('totalWeight').innerText = `${totalWeight.toFixed(2)} kg`;
            document.getElementById('shapeSvg').innerHTML = svgHtml;
        }

        // Initialize on load
        window.onload = calculateBBS;
    </script>
</body>
</html>
