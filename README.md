<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Makmal Transformasi Isometri</title>
    <!-- Tailwind CSS for Styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & ReactDOM -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX compilation in browser -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Phosphor Icons -->
    <script src="https://unpkg.com/@phosphor-icons/web"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body { font-family: 'Inter', Tahoma, Geneva, Verdana, sans-serif; background-color: #f8fafc; }
        
        /* Cursor styling based on action */
        .grid-bg { background-color: white; cursor: crosshair; touch-action: none; }
        .grid-bg.dragging { cursor: grabbing !important; }
        
        /* Custom Range Slider */
        input[type=range] { 
            -webkit-appearance: none; width: 100%; background: transparent; 
        }
        input[type=range]::-webkit-slider-thumb {
            -webkit-appearance: none; height: 16px; width: 16px; border-radius: 50%;
            background: #3b82f6; cursor: pointer; margin-top: -6px; box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }
        input[type=range]::-webkit-slider-runnable-track {
            width: 100%; height: 6px; cursor: pointer; background: #cbd5e1; border-radius: 4px;
        }
        input[type=range]:focus { outline: none; }
        
        /* Prevent selection when clicking fast */
        svg { user-select: none; }
        
        /* Custom Scrollbar */
        ::-webkit-scrollbar { width: 8px; height: 8px; }
        ::-webkit-scrollbar-track { background: #f1f5f9; border-radius: 4px; }
        ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #94a3b8; }

        /* Loading Animation */
        .loader {
            border: 2px solid rgba(255,255,255,0.3);
            border-top: 2px solid #ffffff;
            border-radius: 50%;
            width: 16px;
            height: 16px;
            animation: spin 1s linear infinite;
            display: inline-block;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo, useRef, useCallback } = React;

        const SVG_W = 800; // Resolusi dalaman SVG
        const SVG_H = 800;

        const App = () => {
            // --- State Kamera (Pandangan Infiniti) ---
            const [pan, setPan] = useState({ x: 0, y: 0 }); // Posisi kamera
            const [zoom, setZoom] = useState(25); // Piksel per unit grid
            const [isDraggingCanvas, setIsDraggingCanvas] = useState(false);

            // --- State untuk Bentuk/Objek (senarai titik {x,y}) ---
            const [vertices, setVertices] = useState([
                { x: 2, y: 3 },
                { x: 6, y: 3 },
                { x: 4, y: 7 }
            ]);

            // --- State Transformasi ---
            const [transType, setTransType] = useState('translasi'); 
            const [transParams, setTransParams] = useState({ dx: 4, dy: -5 });
            const [reflParams, setReflParams] = useState({ type: 'x-axis', val: 0 }); 
            const [rotParams, setRotParams] = useState({ cx: 0, cy: 0, angle: 90, dir: 'cw' }); 

            // --- State Animasi ---
            const [isAnimating, setIsAnimating] = useState(false);
            const [progress, setProgress] = useState(1); 

            // --- State Penjana AI ---
            const [promptText, setPromptText] = useState("");
            const [isGenerating, setIsGenerating] = useState(false);
            const [systemMessage, setSystemMessage] = useState(null);

            // Rujukan DOM
            const svgRef = useRef(null);
            const dragStart = useRef({ x: 0, y: 0 });
            const lastPan = useRef({ x: 0, y: 0 });

            // --- Helpers Koordinat Dinamik ---
            const mathToSvgX = useCallback((x) => (SVG_W / 2) + pan.x + (x * zoom), [pan.x, zoom]);
            const mathToSvgY = useCallback((y) => (SVG_H / 2) + pan.y - (y * zoom), [pan.y, zoom]); // Y terbalik
            const svgToMathX = useCallback((svgX) => (svgX - pan.x - (SVG_W / 2)) / zoom, [pan.x, zoom]);
            const svgToMathY = useCallback((svgY) => ((SVG_H / 2) + pan.y - svgY) / zoom, [pan.y, zoom]);

            // Mengira sempadan grid yang kelihatan di skrin
            const visibleBounds = useMemo(() => {
                const minX = Math.floor(svgToMathX(0));
                const maxX = Math.ceil(svgToMathX(SVG_W));
                const minY = Math.floor(svgToMathY(SVG_H));
                const maxY = Math.ceil(svgToMathY(0));
                return { minX, maxX, minY, maxY };
            }, [svgToMathX, svgToMathY]);

            // Menentukan selang (step) grid berdasarkan level zoom supaya nombor tak serabut
            const gridStep = useMemo(() => {
                if (zoom > 40) return 1;
                if (zoom > 15) return 2;
                if (zoom > 5) return 5;
                return 10;
            }, [zoom]);

            // --- Kawalan Tetikus / Sentuh untuk Seret (Pan) & Zum ---
            const handlePointerDown = (e) => {
                if (isAnimating) return;
                dragStart.current = { x: e.clientX, y: e.clientY };
                lastPan.current = { ...pan };
                setIsDraggingCanvas(true);
                e.target.setPointerCapture(e.pointerId);
            };

            const handlePointerMove = (e) => {
                if (!isDraggingCanvas) return;
                const dx = e.clientX - dragStart.current.x;
                const dy = e.clientY - dragStart.current.y;
                
                // Darab dengan nisbah supaya kelajuan seret sama dengan pergerakan SVG
                const rect = svgRef.current.getBoundingClientRect();
                const scaleRatio = SVG_W / rect.width;
                
                setPan({ 
                    x: lastPan.current.x + (dx * scaleRatio), 
                    y: lastPan.current.y + (dy * scaleRatio) 
                });
            };

            const handlePointerUp = (e) => {
                if (!isDraggingCanvas) return;
                setIsDraggingCanvas(false);
                e.target.releasePointerCapture(e.pointerId);

                // Kenal pasti jika ia adalah "Klik" atau "Seret"
                const dx = e.clientX - dragStart.current.x;
                const dy = e.clientY - dragStart.current.y;
                
                if (Math.abs(dx) < 3 && Math.abs(dy) < 3) {
                    // Ia adalah klik - tambah titik pada poligon
                    const rect = svgRef.current.getBoundingClientRect();
                    const scaleX = SVG_W / rect.width;
                    const scaleY = SVG_H / rect.height;
                    const svgX = (e.clientX - rect.left) * scaleX;
                    const svgY = (e.clientY - rect.top) * scaleY;

                    const mathX = Math.round(svgToMathX(svgX));
                    const mathY = Math.round(svgToMathY(svgY));

                    setVertices(prev => [...prev, { x: mathX, y: mathY }]);
                    setProgress(1); 
                }
            };

            // Menghalang skrol halaman semasa zoom dalam satah
            useEffect(() => {
                const svgElement = svgRef.current;
                const handleWheel = (e) => {
                    e.preventDefault();
                    const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
                    setZoom(prev => Math.max(2, Math.min(100, prev * zoomFactor)));
                };
                if (svgElement) {
                    svgElement.addEventListener('wheel', handleWheel, { passive: false });
                }
                return () => {
                    if (svgElement) svgElement.removeEventListener('wheel', handleWheel);
                };
            }, []);

            // Kawalan Butang UI
            const zoomIn = () => setZoom(z => Math.min(100, z * 1.2));
            const zoomOut = () => setZoom(z => Math.max(2, z / 1.2));
            const resetView = () => { setPan({x: 0, y: 0}); setZoom(25); };

            // --- Logik Penjanaan AI (Gemini API) ---
            const handleGenerateAI = async () => {
                if (!promptText.trim()) return;
                setIsGenerating(true); setSystemMessage(null);
                const apiKey = ""; 
                const systemPrompt = `Anda adalah penjana koordinat pintar untuk satah Cartes.
Tugas anda: Tukar permintaan teks pengguna kepada senarai koordinat bucu (vertices) untuk membentuk poligon tersebut.
Syarat WAJIB:
1. Koordinat 'x' dan 'y' mestilah nombor bulat (integer) antara -20 hingga 20.
2. Mesti kembalikan sekurang-kurangnya 3 titik.
3. Susun titik-titik tersebut secara berurutan (arah jam atau lawan jam) supaya apabila garisan disambung, ia membentuk bentuk yang betul dan garisan tidak bersilang.
4. JAWAPAN MESTI HANYA ARRAY JSON YANG SAH. Jangan tulis teks penerangan. HANYA array seperti ini:
[{"x": 0, "y": 2}, {"x": 3, "y": -1}, {"x": -3, "y": -1}]`;

                const payload = {
                    contents: [{ parts: [{ text: promptText }] }],
                    systemInstruction: { parts: [{ text: systemPrompt }] },
                    generationConfig: { responseMimeType: "application/json" }
                };

                let result = null;
                let retryCount = 0;
                let success = false;
                const delays = [1000, 2000, 4000, 8000];

                while (retryCount <= 4 && !success) {
                    try {
                        if (retryCount > 0) await new Promise(r => setTimeout(r, delays[retryCount - 1]));
                        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                            method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload)
                        });
                        if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
                        result = await response.json();
                        success = true;
                    } catch (error) { retryCount++; }
                }

                if (success && result) {
                    try {
                        const textResponse = result.candidates?.[0]?.content?.parts?.[0]?.text;
                        if (textResponse) {
                            let cleanJson = textResponse.trim().replace(/```json/g, '').replace(/```/g, '').trim();
                            const points = JSON.parse(cleanJson);
                            if (Array.isArray(points) && points.length >= 3) {
                                setVertices(points); setProgress(1); setPromptText("");
                                setSystemMessage({ type: 'success', text: 'Bentuk berjaya dijana oleh AI!' });
                                // Reset view to ensure shape is visible if it's large
                                resetView();
                            } else {
                                setSystemMessage({ type: 'error', text: 'AI tidak dapat menghasilkan poligon yang sah.' });
                            }
                        }
                    } catch (e) {
                        setSystemMessage({ type: 'error', text: 'Ralat menterjemah maklumat bentuk dari AI.' });
                    }
                } else {
                    setSystemMessage({ type: 'error', text: 'Gagal menyambung ke pelayan AI.' });
                }
                setIsGenerating(false);
            };

            useEffect(() => {
                if (systemMessage) {
                    const timer = setTimeout(() => setSystemMessage(null), 5000);
                    return () => clearTimeout(timer);
                }
            }, [systemMessage]);

            // --- Fungsi Transformasi Matematik ---
            const calculateTranslation = (pt, params, prog = 1) => ({ x: pt.x + params.dx * prog, y: pt.y + params.dy * prog });
            const calculateReflection = (pt, params, prog = 1) => {
                let targetX = pt.x; let targetY = pt.y;
                if (params.type === 'x-axis') targetY = -pt.y;
                else if (params.type === 'y-axis') targetX = -pt.x;
                else if (params.type === 'y-line') targetY = 2 * params.val - pt.y;
                else if (params.type === 'x-line') targetX = 2 * params.val - pt.x;
                return { x: pt.x + (targetX - pt.x) * prog, y: pt.y + (targetY - pt.y) * prog };
            };
            const calculateRotation = (pt, params, prog = 1) => {
                const { cx, cy, angle, dir } = params;
                let rad = (dir === 'cw' ? -angle : angle) * prog * (Math.PI / 180);
                let tempX = pt.x - cx; let tempY = pt.y - cy;
                return {
                    x: (tempX * Math.cos(rad) - tempY * Math.sin(rad)) + cx,
                    y: (tempX * Math.sin(rad) + tempY * Math.cos(rad)) + cy
                };
            };

            const currentPoints = useMemo(() => {
                if (vertices.length === 0) return [];
                return vertices.map(pt => {
                    if (transType === 'translasi') return calculateTranslation(pt, transParams, progress);
                    if (transType === 'pantulan') return calculateReflection(pt, reflParams, progress);
                    if (transType === 'putaran') return calculateRotation(pt, rotParams, progress);
                    return pt;
                });
            }, [vertices, transType, transParams, reflParams, rotParams, progress]);

            const finalPoints = useMemo(() => {
                if (vertices.length === 0) return [];
                return vertices.map(pt => {
                    if (transType === 'translasi') return calculateTranslation(pt, transParams, 1);
                    if (transType === 'pantulan') return calculateReflection(pt, reflParams, 1);
                    if (transType === 'putaran') return calculateRotation(pt, rotParams, 1);
                    return pt;
                });
            }, [vertices, transType, transParams, reflParams, rotParams]);

            // --- Logik Animasi ---
            const startAnimation = () => {
                if (vertices.length === 0) return;
                setIsAnimating(true); setProgress(0);
            };

            useEffect(() => {
                let animationFrameId; let startTimestamp = null; const DURATION = 1500; 
                const step = (timestamp) => {
                    if (!startTimestamp) startTimestamp = timestamp;
                    let p = (timestamp - startTimestamp) / DURATION;
                    if (p >= 1) { p = 1; setIsAnimating(false); }
                    setProgress(p);
                    if (p < 1) animationFrameId = window.requestAnimationFrame(step);
                };
                if (isAnimating && progress < 1) animationFrameId = window.requestAnimationFrame(step);
                return () => { if (animationFrameId) window.cancelAnimationFrame(animationFrameId); };
            }, [isAnimating]);

            const clearVertices = () => { setVertices([]); setProgress(1); };
            const setPresetShape = (shape) => {
                if (shape === 'segitiga') setVertices([{ x: 2, y: 2 }, { x: 7, y: 2 }, { x: 2, y: 6 }]);
                if (shape === 'segiempat') setVertices([{ x: 2, y: 2 }, { x: 6, y: 2 }, { x: 6, y: 5 }, { x: 2, y: 5 }]);
                setProgress(1); resetView();
            };

            const makePolygonPath = (points) => {
                if (!points || points.length === 0) return "";
                const svgPts = points.map(p => `${mathToSvgX(p.x)},${mathToSvgY(p.y)}`);
                return `M ${svgPts.join(' L ')} Z`;
            };

            // --- Render Grid Dinamik ---
            const renderDynamicGrid = () => {
                const lines = [];
                const { minX, maxX, minY, maxY } = visibleBounds;
                
                // Cari gandaan terdekat untuk start
                const startX = Math.floor(minX / gridStep) * gridStep;
                const startY = Math.floor(minY / gridStep) * gridStep;

                // Garisan Menegak (X)
                for (let i = startX; i <= maxX; i += gridStep) {
                    const svgX = mathToSvgX(i);
                    const isAxis = i === 0;
                    lines.push(
                        <g key={`x-${i}`}>
                            <line x1={svgX} y1="0" x2={svgX} y2={SVG_H} stroke={isAxis ? "#334155" : "#f1f5f9"} strokeWidth={isAxis ? "2" : "1"} />
                            {!isAxis && (
                                <text x={svgX} y={mathToSvgY(0) + 15} fontSize="10" textAnchor="middle" fill="#94a3b8" fontWeight="500">{i}</text>
                            )}
                        </g>
                    );
                }

                // Garisan Mengufuk (Y)
                for (let i = startY; i <= maxY; i += gridStep) {
                    const svgY = mathToSvgY(i);
                    const isAxis = i === 0;
                    lines.push(
                        <g key={`y-${i}`}>
                            <line x1="0" y1={svgY} x2={SVG_W} y2={svgY} stroke={isAxis ? "#334155" : "#f1f5f9"} strokeWidth={isAxis ? "2" : "1"} />
                            {!isAxis && (
                                <text x={mathToSvgX(0) - 15} y={svgY + 4} fontSize="10" textAnchor="middle" fill="#94a3b8" fontWeight="500">{i}</text>
                            )}
                        </g>
                    );
                }
                
                // Label Asalan (0)
                lines.push(<text key="zero" x={mathToSvgX(0) - 12} y={mathToSvgY(0) + 16} fontSize="12" fontWeight="bold" fill="#334155">0</text>);
                
                return lines;
            };

            // --- Render Visual Aids ---
            const renderVisualAids = () => {
                if (progress < 1) return null; 
                let aids = [];

                if (transType === 'pantulan') {
                    let axisLine = null;
                    if (reflParams.type === 'x-axis') {
                        axisLine = <line x1="0" y1={mathToSvgY(0)} x2={SVG_W} y2={mathToSvgY(0)} stroke="#16a34a" strokeWidth="3" strokeDasharray="6,6" key="axis" />;
                    } else if (reflParams.type === 'y-axis') {
                        axisLine = <line x1={mathToSvgX(0)} y1="0" x2={mathToSvgX(0)} y2={SVG_H} stroke="#16a34a" strokeWidth="3" strokeDasharray="6,6" key="axis" />;
                    } else if (reflParams.type === 'x-line') {
                        axisLine = <line x1={mathToSvgX(reflParams.val)} y1="0" x2={mathToSvgX(reflParams.val)} y2={SVG_H} stroke="#16a34a" strokeWidth="3" strokeDasharray="6,6" key="axis" />;
                    } else if (reflParams.type === 'y-line') {
                        axisLine = <line x1="0" y1={mathToSvgY(reflParams.val)} x2={SVG_W} y2={mathToSvgY(reflParams.val)} stroke="#16a34a" strokeWidth="3" strokeDasharray="6,6" key="axis" />;
                    }
                    aids.push(axisLine);

                    vertices.forEach((pt, i) => {
                        let fPt = finalPoints[i];
                        aids.push(<line key={`aid-${i}`} x1={mathToSvgX(pt.x)} y1={mathToSvgY(pt.y)} x2={mathToSvgX(fPt.x)} y2={mathToSvgY(fPt.y)} stroke="rgba(34, 197, 94, 0.3)" strokeWidth="2" strokeDasharray="4,4" />);
                    });
                }
                else if (transType === 'translasi') {
                    vertices.forEach((pt, i) => {
                        let fPt = finalPoints[i];
                        aids.push(
                            <line key={`aid-${i}`} x1={mathToSvgX(pt.x)} y1={mathToSvgY(pt.y)} x2={mathToSvgX(fPt.x)} y2={mathToSvgY(fPt.y)} stroke="rgba(168, 85, 247, 0.4)" strokeWidth="2" strokeDasharray="5,5" />
                        );
                        aids.push(
                            <circle key={`dot-${i}`} cx={mathToSvgX(fPt.x)} cy={mathToSvgY(fPt.y)} r="3" fill="#9333ea" />
                        );
                    });
                }
                else if (transType === 'putaran') {
                    aids.push(<circle key="center" cx={mathToSvgX(rotParams.cx)} cy={mathToSvgY(rotParams.cy)} r="5" fill="#ea580c" />);
                    aids.push(<text key="ctext" x={mathToSvgX(rotParams.cx) + 8} y={mathToSvgY(rotParams.cy) - 8} fill="#ea580c" fontSize="12" fontWeight="bold">Pusat Putaran</text>);
                    
                    if (vertices.length > 0) {
                        let pt = vertices[0];
                        let fPt = finalPoints[0];
                        aids.push(<line key="rad1" x1={mathToSvgX(rotParams.cx)} y1={mathToSvgY(rotParams.cy)} x2={mathToSvgX(pt.x)} y2={mathToSvgY(pt.y)} stroke="rgba(234, 88, 12, 0.4)" strokeWidth="2" strokeDasharray="5,5" />);
                        aids.push(<line key="rad2" x1={mathToSvgX(rotParams.cx)} y1={mathToSvgY(rotParams.cy)} x2={mathToSvgX(fPt.x)} y2={mathToSvgY(fPt.y)} stroke="rgba(234, 88, 12, 0.4)" strokeWidth="2" strokeDasharray="5,5" />);
                    }
                }
                return aids;
            };

            const generateExplanation = () => {
                if (vertices.length === 0) return (
                    <div className="flex items-center justify-center h-32 text-slate-400 italic">
                        <i className="ph ph-info mr-2 text-xl"></i> Sila lukis atau jana bentuk pada satah Cartes dahulu.
                    </div>
                );

                let explanation = [];
                let isometriText = "Transformasi ini adalah suatu isometri kerana bentuk dan saiz imej adalah sama dengan objek (kekongruenan dikekalkan).";

                if (transType === 'translasi') {
                    explanation.push(<p key="desc" className="text-lg font-bold mb-4 text-slate-800">Translasi dengan vektor <span className="text-purple-600 bg-purple-50 px-2 py-1 rounded">( {transParams.dx} , {transParams.dy} )</span></p>);
                    vertices.forEach((pt, i) => {
                        let finalPt = finalPoints[i];
                        let label = String.fromCharCode(65 + i % 26); 
                        explanation.push(
                            <div key={i} className="flex items-center gap-3 font-mono text-sm mb-2 bg-slate-50 p-2 rounded border border-slate-100">
                                <span className="text-blue-600 font-bold">{label}({pt.x}, {pt.y})</span>
                                <i className="ph ph-arrow-right text-slate-400"></i>
                                <span className="text-red-500 font-bold">{label}'({finalPt.x.toFixed(1)}, {finalPt.y.toFixed(1)})</span>
                            </div>
                        );
                    });
                } 
                else if (transType === 'pantulan') {
                    let pDesc = reflParams.type === 'x-axis' ? "Paksi-x (y = 0)" : reflParams.type === 'y-axis' ? "Paksi-y (x = 0)" : reflParams.type === 'x-line' ? `Garis x = ${reflParams.val}` : `Garis y = ${reflParams.val}`;
                    explanation.push(<p key="desc" className="text-lg font-bold mb-4 text-slate-800">Pantulan pada <span className="text-green-600 bg-green-50 px-2 py-1 rounded">{pDesc}</span></p>);
                    vertices.forEach((pt, i) => {
                        let finalPt = finalPoints[i]; let label = String.fromCharCode(65 + i % 26);
                        explanation.push(
                            <div key={i} className="flex items-center gap-3 font-mono text-sm mb-2 bg-slate-50 p-2 rounded border border-slate-100">
                                <span className="text-blue-600 font-bold">{label}({pt.x}, {pt.y})</span>
                                <i className="ph ph-arrow-right text-slate-400"></i>
                                <span className="text-red-500 font-bold">{label}'({finalPt.x.toFixed(1)}, {finalPt.y.toFixed(1)})</span>
                            </div>
                        );
                    });
                }
                else if (transType === 'putaran') {
                    let dirDesc = rotParams.dir === 'cw' ? "Ikut Jam" : "Lawan Jam";
                    explanation.push(<p key="desc" className="text-lg font-bold mb-4 text-slate-800">Putaran <span className="text-orange-600 bg-orange-50 px-2 py-1 rounded">{rotParams.angle}&deg; {dirDesc}</span> di pusat <span className="text-orange-600 bg-orange-50 px-2 py-1 rounded">({rotParams.cx}, {rotParams.cy})</span></p>);
                    vertices.forEach((pt, i) => {
                        let finalPt = finalPoints[i]; let label = String.fromCharCode(65 + i % 26);
                        explanation.push(
                            <div key={i} className="flex items-center gap-3 font-mono text-sm mb-2 bg-slate-50 p-2 rounded border border-slate-100">
                                <span className="text-blue-600 font-bold">{label}({pt.x}, {pt.y})</span>
                                <i className="ph ph-arrow-right text-slate-400"></i>
                                <span className="text-red-500 font-bold">{label}'({finalPt.x.toFixed(1)}, {finalPt.y.toFixed(1)})</span>
                            </div>
                        );
                    });
                }

                return (
                    <div className="animate-fade-in grid grid-cols-1 md:grid-cols-2 gap-6">
                        <div>
                            <h3 className="text-sm font-semibold uppercase tracking-wider text-slate-500 mb-3">Pemetaan Titik Koordinat</h3>
                            <div className="max-h-64 overflow-y-auto pr-2 custom-scrollbar">{explanation}</div>
                        </div>
                        <div>
                            <h3 className="text-sm font-semibold uppercase tracking-wider text-slate-500 mb-3">Sifat Transformasi</h3>
                            <div className="p-4 bg-indigo-50 border border-indigo-100 rounded-xl flex items-start gap-3 text-indigo-900 shadow-sm">
                                <i className="ph-fill ph-info text-2xl text-indigo-500"></i>
                                <p className="text-sm leading-relaxed">{isometriText}</p>
                            </div>
                        </div>
                    </div>
                );
            };

            return (
                <div className="min-h-screen flex flex-col">
                    <header className="bg-gradient-to-r from-blue-700 via-indigo-700 to-purple-800 text-white p-5 shadow-lg relative overflow-hidden">
                        <div className="absolute top-0 right-0 -mr-16 -mt-16 w-64 h-64 rounded-full bg-white opacity-10 blur-3xl"></div>
                        <div className="container mx-auto flex flex-col sm:flex-row justify-between items-center relative z-10 gap-3">
                            <h1 className="text-2xl sm:text-3xl font-bold flex items-center gap-3 tracking-tight">
                                <div className="bg-white/20 p-2 rounded-lg backdrop-blur-sm"><i className="ph ph-math-operations"></i></div>
                                Makmal Transformasi Isometri
                            </h1>
                            <span className="bg-white text-indigo-800 px-4 py-1.5 rounded-full text-sm font-bold shadow-sm flex items-center gap-2">
                                <i className="ph-fill ph-infinity"></i> Satah Infiniti
                            </span>
                        </div>
                    </header>

                    <main className="flex-1 container mx-auto p-4 lg:p-6 grid grid-cols-1 lg:grid-cols-12 gap-6 items-start">
                        
                        {/* PANEL KIRI (Kawalan) */}
                        <div className="lg:col-span-4 flex flex-col gap-6">
                            
                            {/* Kad 1: Bina Objek Asal */}
                            <div className="bg-white p-6 rounded-2xl shadow-sm border border-slate-200">
                                <div className="flex items-center gap-2 mb-5">
                                    <div className="w-8 h-8 rounded-full bg-blue-100 flex items-center justify-center text-blue-600"><i className="ph-fill ph-shapes text-lg"></i></div>
                                    <h2 className="text-lg font-bold text-slate-800">1. Bina Objek Asal</h2>
                                </div>
                                
                                <div className="mb-5 bg-gradient-to-br from-purple-50 to-fuchsia-50 p-4 rounded-xl border border-purple-100 relative overflow-hidden">
                                    <label className="block text-sm font-bold text-purple-900 mb-2 flex items-center gap-1.5"><i className="ph-fill ph-magic-wand"></i> Jana Bentuk dengan AI</label>
                                    <div className="flex gap-2 relative z-10">
                                        <input type="text" value={promptText} onChange={(e) => setPromptText(e.target.value)} onKeyDown={(e) => e.key === 'Enter' && handleGenerateAI()} placeholder="Cth: Lukiskan sebuah bintang" className="flex-1 p-2.5 bg-white border border-purple-200 rounded-lg text-sm focus:ring-2 focus:ring-purple-400 focus:border-purple-400 outline-none shadow-sm" />
                                        <button onClick={handleGenerateAI} disabled={isGenerating || !promptText.trim()} className={`px-4 py-2.5 rounded-lg text-sm font-bold text-white shadow-sm ${isGenerating || !promptText.trim() ? 'bg-purple-300 cursor-not-allowed' : 'bg-purple-600 hover:bg-purple-700 active:scale-95'}`}>
                                            {isGenerating ? <span className="loader"></span> : 'Jana'}
                                        </button>
                                    </div>
                                </div>

                                {systemMessage && (
                                    <div className={`p-3 mb-5 text-sm rounded-lg border flex items-start gap-2 shadow-sm ${systemMessage.type === 'error' ? 'bg-red-50 text-red-700 border-red-200' : 'bg-emerald-50 text-emerald-700 border-emerald-200'}`}>
                                        <i className={`mt-0.5 ${systemMessage.type === 'error' ? 'ph-fill ph-warning-circle' : 'ph-fill ph-check-circle'}`}></i>
                                        <span className="font-medium">{systemMessage.text}</span>
                                    </div>
                                )}

                                <div className="space-y-3">
                                    <label className="block text-xs font-semibold uppercase tracking-wider text-slate-500">Bentuk Sedia Ada</label>
                                    <div className="grid grid-cols-2 gap-3">
                                        <button onClick={() => setPresetShape('segitiga')} className="bg-slate-50 hover:bg-blue-50 border border-slate-200 hover:border-blue-200 p-2.5 rounded-lg text-sm font-medium text-slate-700 flex justify-center items-center gap-2"><i className="ph-fill ph-triangle"></i> Segi Tiga</button>
                                        <button onClick={() => setPresetShape('segiempat')} className="bg-slate-50 hover:bg-blue-50 border border-slate-200 hover:border-blue-200 p-2.5 rounded-lg text-sm font-medium text-slate-700 flex justify-center items-center gap-2"><i className="ph-fill ph-square"></i> Segi Empat</button>
                                    </div>
                                </div>
                                
                                <button onClick={clearVertices} className="mt-4 w-full border border-red-200 text-red-600 hover:bg-red-50 hover:border-red-300 p-2.5 rounded-lg text-sm font-bold flex justify-center items-center gap-2">
                                    <i className="ph-bold ph-trash"></i> Kosongkan Grid
                                </button>
                            </div>

                            {/* Kad 2: Tetapan Transformasi */}
                            <div className="bg-white p-6 rounded-2xl shadow-sm border border-slate-200">
                                <div className="flex items-center gap-2 mb-5">
                                    <div className="w-8 h-8 rounded-full bg-indigo-100 flex items-center justify-center text-indigo-600"><i className="ph-fill ph-arrows-out-cardinal text-lg"></i></div>
                                    <h2 className="text-lg font-bold text-slate-800">2. Tetapan Transformasi</h2>
                                </div>
                                
                                <div className="flex bg-slate-100 p-1 rounded-xl mb-6">
                                    {['translasi', 'pantulan', 'putaran'].map(type => (
                                        <button key={type} onClick={() => {setTransType(type); setProgress(1);}} className={`flex-1 py-2 text-sm font-bold rounded-lg capitalize transition-all ${transType === type ? 'bg-white text-indigo-700 shadow' : 'text-slate-500 hover:text-slate-700 hover:bg-slate-50'}`}>{type}</button>
                                    ))}
                                </div>

                                <div className="min-h-[140px]">
                                    {transType === 'translasi' && (
                                        <div className="space-y-5">
                                            <div className="bg-slate-50 p-4 rounded-xl border border-slate-100">
                                                <div className="flex justify-between items-center mb-2">
                                                    <label className="text-sm font-semibold text-slate-700">Pergerakan Paksi-x (dx)</label>
                                                    <input type="number" className="w-16 p-1 border rounded text-center text-sm font-bold text-indigo-700" value={transParams.dx} onChange={e => {setTransParams({...transParams, dx: parseInt(e.target.value)||0}); setProgress(1);}} />
                                                </div>
                                                <input type="range" min="-20" max="20" value={transParams.dx} onChange={e => {setTransParams({...transParams, dx: parseInt(e.target.value)}); setProgress(1);}} />
                                            </div>
                                            <div className="bg-slate-50 p-4 rounded-xl border border-slate-100">
                                                <div className="flex justify-between items-center mb-2">
                                                    <label className="text-sm font-semibold text-slate-700">Pergerakan Paksi-y (dy)</label>
                                                    <input type="number" className="w-16 p-1 border rounded text-center text-sm font-bold text-indigo-700" value={transParams.dy} onChange={e => {setTransParams({...transParams, dy: parseInt(e.target.value)||0}); setProgress(1);}} />
                                                </div>
                                                <input type="range" min="-20" max="20" value={transParams.dy} onChange={e => {setTransParams({...transParams, dy: parseInt(e.target.value)}); setProgress(1);}} />
                                            </div>
                                        </div>
                                    )}

                                    {transType === 'pantulan' && (
                                        <div className="space-y-5">
                                            <div>
                                                <label className="block text-sm font-semibold text-slate-700 mb-2">Pilih Paksi Pantulan:</label>
                                                <div className="relative">
                                                    <select className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl text-sm font-medium focus:ring-2 focus:ring-indigo-500 appearance-none outline-none" value={reflParams.type} onChange={e => {setReflParams({...reflParams, type: e.target.value}); setProgress(1);}}>
                                                        <option value="x-axis">Paksi-x (y = 0)</option>
                                                        <option value="y-axis">Paksi-y (x = 0)</option>
                                                        <option value="x-line">Garis Menegak (x = a)</option>
                                                        <option value="y-line">Garis Mengufuk (y = b)</option>
                                                    </select>
                                                    <i className="ph-bold ph-caret-down absolute right-4 top-3.5 text-slate-400 pointer-events-none"></i>
                                                </div>
                                            </div>
                                            {(reflParams.type === 'x-line' || reflParams.type === 'y-line') && (
                                                <div className="bg-slate-50 p-4 rounded-xl border border-slate-100">
                                                    <div className="flex justify-between items-center mb-2">
                                                        <label className="text-sm font-semibold text-slate-700">Nilai paksi ({reflParams.type === 'x-line' ? 'x' : 'y'})</label>
                                                        <input type="number" className="w-16 p-1 border rounded text-center text-sm font-bold text-indigo-700" value={reflParams.val} onChange={e => {setReflParams({...reflParams, val: parseInt(e.target.value)||0}); setProgress(1);}} />
                                                    </div>
                                                    <input type="range" min="-20" max="20" value={reflParams.val} onChange={e => {setReflParams({...reflParams, val: parseInt(e.target.value)}); setProgress(1);}} />
                                                </div>
                                            )}
                                        </div>
                                    )}

                                    {transType === 'putaran' && (
                                        <div className="space-y-4">
                                            <div className="grid grid-cols-2 gap-4">
                                                <div>
                                                    <label className="block text-xs font-semibold uppercase text-slate-500 mb-1.5">Pusat X</label>
                                                    <input type="number" className="w-full p-2.5 bg-slate-50 border border-slate-200 rounded-lg text-sm font-bold text-center focus:ring-2 focus:ring-indigo-500 outline-none" value={rotParams.cx} onChange={e => {setRotParams({...rotParams, cx: parseInt(e.target.value)||0}); setProgress(1);}} />
                                                </div>
                                                <div>
                                                    <label className="block text-xs font-semibold uppercase text-slate-500 mb-1.5">Pusat Y</label>
                                                    <input type="number" className="w-full p-2.5 bg-slate-50 border border-slate-200 rounded-lg text-sm font-bold text-center focus:ring-2 focus:ring-indigo-500 outline-none" value={rotParams.cy} onChange={e => {setRotParams({...rotParams, cy: parseInt(e.target.value)||0}); setProgress(1);}} />
                                                </div>
                                            </div>
                                            <div>
                                                <label className="block text-xs font-semibold uppercase text-slate-500 mb-2">Sudut Putaran</label>
                                                <div className="flex gap-2">
                                                    {[90, 180, 270].map(deg => (
                                                        <button key={deg} onClick={() => {setRotParams({...rotParams, angle: deg}); setProgress(1);}} className={`flex-1 py-2 rounded-lg text-sm transition-colors ${rotParams.angle === deg ? 'bg-indigo-100 border-2 border-indigo-500 text-indigo-700 font-bold' : 'bg-slate-50 border-2 border-transparent text-slate-600 hover:bg-slate-100'}`}>{deg}&deg;</button>
                                                    ))}
                                                </div>
                                            </div>
                                            <div>
                                                <label className="block text-xs font-semibold uppercase text-slate-500 mb-2">Arah Putaran</label>
                                                <div className="flex gap-2 bg-slate-50 p-1 rounded-lg">
                                                    <button onClick={() => {setRotParams({...rotParams, dir: 'cw'}); setProgress(1);}} className={`flex-1 py-2 rounded-md text-xs font-bold transition-all ${rotParams.dir === 'cw' ? 'bg-white shadow-sm text-indigo-700' : 'text-slate-500 hover:text-slate-700'}`}>Ikut Jam</button>
                                                    <button onClick={() => {setRotParams({...rotParams, dir: 'ccw'}); setProgress(1);}} className={`flex-1 py-2 rounded-md text-xs font-bold transition-all ${rotParams.dir === 'ccw' ? 'bg-white shadow-sm text-indigo-700' : 'text-slate-500 hover:text-slate-700'}`}>Lawan Jam</button>
                                                </div>
                                            </div>
                                        </div>
                                    )}
                                </div>

                                <button onClick={startAnimation} disabled={vertices.length === 0 || isAnimating}
                                    className={`mt-6 w-full py-3.5 rounded-xl font-bold text-white transition-all transform active:scale-95 flex justify-center items-center gap-2 ${vertices.length === 0 ? 'bg-slate-300 cursor-not-allowed' : 'bg-emerald-500 hover:bg-emerald-600 shadow-md'}`}>
                                    <i className={`ph-fill ${isAnimating ? 'ph-spinner animate-spin' : 'ph-play-circle'} text-xl`}></i>
                                    {isAnimating ? 'Sedang Memainkan...' : 'Mainkan Animasi'}
                                </button>
                            </div>
                        </div>

                        {/* PANEL KANAN (Satah & Analisis) */}
                        <div className="lg:col-span-8 flex flex-col gap-6">
                            
                            <div className="bg-white p-4 lg:p-6 rounded-2xl shadow-sm border border-slate-200 relative">
                                {/* Petunjuk Interaksi */}
                                <div className="absolute top-8 left-8 z-10 bg-white/90 backdrop-blur px-3 py-2 rounded-xl border border-slate-200 shadow-sm text-xs text-slate-600 flex flex-col gap-1 pointer-events-none">
                                    <p className="font-bold text-indigo-600 flex items-center gap-1"><i className="ph-fill ph-grid-four"></i> Satah Infiniti</p>
                                    <p>👆 <b>Klik</b> untuk lakar bucu</p>
                                    <p>✋ <b>Seret</b> (drag) untuk gerak satah</p>
                                    <p>🖱️ <b>Tatal</b> (scroll) untuk zum</p>
                                </div>

                                {/* Alat Navigasi Satah */}
                                <div className="absolute bottom-8 right-8 z-10 flex flex-col gap-2 bg-white/90 backdrop-blur p-2 rounded-xl border border-slate-200 shadow-sm">
                                    <button onClick={zoomIn} title="Zum Masuk" className="w-10 h-10 rounded-lg bg-slate-50 hover:bg-indigo-50 text-slate-700 hover:text-indigo-600 flex justify-center items-center transition"><i className="ph-bold ph-plus text-lg"></i></button>
                                    <button onClick={zoomOut} title="Zum Keluar" className="w-10 h-10 rounded-lg bg-slate-50 hover:bg-indigo-50 text-slate-700 hover:text-indigo-600 flex justify-center items-center transition"><i className="ph-bold ph-minus text-lg"></i></button>
                                    <div className="w-full h-px bg-slate-200 my-1"></div>
                                    <button onClick={resetView} title="Kembali ke Pusat" className="w-10 h-10 rounded-lg bg-indigo-500 hover:bg-indigo-600 text-white flex justify-center items-center shadow transition"><i className="ph-bold ph-house text-lg"></i></button>
                                </div>

                                {isGenerating && (
                                    <div className="absolute inset-0 z-20 bg-white/70 flex flex-col justify-center items-center backdrop-blur-sm rounded-2xl pointer-events-none">
                                        <div className="w-16 h-16 border-4 border-indigo-100 border-t-indigo-600 rounded-full animate-spin shadow-lg"></div>
                                        <p className="mt-4 font-bold text-indigo-800 bg-white px-4 py-1 rounded-full shadow-sm">AI sedang mengira titik...</p>
                                    </div>
                                )}

                                <div className="w-full aspect-square md:aspect-video lg:aspect-square flex justify-center overflow-hidden rounded-xl bg-slate-50 border border-slate-200 shadow-inner">
                                    <svg 
                                        ref={svgRef}
                                        width="100%" 
                                        height="100%" 
                                        className={`grid-bg w-full h-full ${isDraggingCanvas ? 'dragging' : ''}`}
                                        viewBox={`0 0 ${SVG_W} ${SVG_H}`}
                                        preserveAspectRatio="xMidYMid slice"
                                        onPointerDown={handlePointerDown}
                                        onPointerMove={handlePointerMove}
                                        onPointerUp={handlePointerUp}
                                        onPointerLeave={handlePointerUp}
                                    >
                                        {/* Grid Line Dynamics */}
                                        {renderDynamicGrid()}

                                        {/* Visual Bantuan */}
                                        {renderVisualAids()}

                                        {/* Lukis Imej Akhir (Merah/Oren) */}
                                        {vertices.length > 0 && progress > 0 && (
                                            <g>
                                                <path d={makePolygonPath(currentPoints)} fill={progress === 1 ? "rgba(239, 68, 68, 0.25)" : "rgba(245, 158, 11, 0.4)"} stroke={progress === 1 ? "#ef4444" : "#f59e0b"} strokeWidth="2.5" strokeLinejoin="round" strokeDasharray={progress === 1 ? "6,4" : ""} />
                                                {currentPoints.map((pt, i) => (
                                                    <g key={`img-pt-${i}`}>
                                                        <circle cx={mathToSvgX(pt.x)} cy={mathToSvgY(pt.y)} r="5" fill="#ef4444" stroke="#fff" strokeWidth="1.5" />
                                                        <text x={mathToSvgX(pt.x) + 8} y={mathToSvgY(pt.y) - 8} fontSize="14" fontWeight="bold" fill="#ef4444" className="drop-shadow-sm">{String.fromCharCode(65 + i % 26)}'</text>
                                                    </g>
                                                ))}
                                            </g>
                                        )}

                                        {/* Lukis Objek Asal (Biru) */}
                                        {vertices.length > 0 && (
                                            <g>
                                                <path d={makePolygonPath(vertices)} fill="rgba(59, 130, 246, 0.3)" stroke="#3b82f6" strokeWidth="2.5" strokeLinejoin="round" />
                                                {vertices.map((pt, i) => (
                                                    <g key={`obj-pt-${i}`}>
                                                        <circle cx={mathToSvgX(pt.x)} cy={mathToSvgY(pt.y)} r="5" fill="#3b82f6" stroke="#fff" strokeWidth="1.5"/>
                                                        <text x={mathToSvgX(pt.x) + 8} y={mathToSvgY(pt.y) - 8} fontSize="14" fontWeight="bold" fill="#1d4ed8" className="drop-shadow-sm">{String.fromCharCode(65 + i % 26)}</text>
                                                    </g>
                                                ))}
                                            </g>
                                        )}
                                    </svg>
                                </div>
                            </div>

                            <div className="bg-white p-6 rounded-2xl shadow-sm border border-slate-200">
                                <div className="flex items-center gap-2 mb-5 border-b border-slate-100 pb-4">
                                    <div className="w-8 h-8 rounded-full bg-emerald-100 flex items-center justify-center text-emerald-600"><i className="ph-fill ph-chart-line-up text-lg"></i></div>
                                    <h2 className="text-lg font-bold text-slate-800">Analisis Perubahan (Penyelesaian)</h2>
                                </div>
                                {generateExplanation()}
                            </div>

                        </div>
                    </main>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
