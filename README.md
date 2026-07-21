[의생명 최종.html](https://github.com/user-attachments/files/30236872/default.html)

<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>타우린 항산화 시뮬레이터</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>
body {
    font-family: "맑은 고딕", sans-serif;
    background: #f5f5f5;
    margin: 0;
}

header {
    background: #1976d2;
    color: white;
    text-align: center;
    padding: 20px;
    font-size: 28px;
    font-weight: bold;
}

.container {
    width: 1200px;
    margin: 30px auto;
    display: grid;
    grid-template-columns: 380px 1fr;
    gap: 25px;
}

.left, .right {
    background: white;
    padding: 25px;
    border-radius: 12px;
    box-shadow: 0px 2px 10px rgba(0,0,0,0.1);
}

h2 { margin-top: 0; }
label { font-weight: bold; }

select, input {
    width: 100%;
    padding: 10px;
    margin-top: 8px;
    margin-bottom: 18px;
    font-size: 16px;
    box-sizing: border-box;
}

button {
    width: 100%;
    padding: 12px;
    font-size: 16px;
    background: #1976d2;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    margin-bottom: 10px;
}

button:hover { background: #0d47a1; }

.result {
    margin-top: 25px;
    padding: 15px;
    background: #eeeeee;
    border-radius: 10px;
    line-height: 1.8;
}

canvas {
    width: 100% !important;
    height: 400px !important;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 25px;
}

th, td {
    border: 1px solid #ccc;
    padding: 8px;
    text-align: center;
}

#analysis {
    margin-top: 15px;
    padding: 15px;
    background: #eef7ff;
    border-radius: 8px;
}
</style>
</head>

<body>

<header>타우린 항산화 시뮬레이터</header>

<div class="container">
    <!-- 왼쪽 영역: 입력 및 컨트롤 -->
    <div class="left">
        <h2>입력 변수</h2>

        <label for="infection">감염 여부</label>
        <select id="infection">
            <option value="0">없음</option>
            <option value="1">있음</option>
        </select>

        <label for="wound">상처 여부</label>
        <select id="wound">
            <option value="0">없음</option>
            <option value="1">있음</option>
        </select>

        <label for="exercise">운동</label>
        <select id="exercise">
            <option value="0">운동 안함</option>
            <option value="1">1~3시간</option>
            <option value="2">3시간 이상</option>
        </select>

        <label for="sleep">수면</label>
        <select id="sleep">
            <option value="0">7시간 이상</option>
            <option value="0.5">4~7시간</option>
            <option value="1">4시간 미만</option>
        </select>

        <label for="taurine">타우린 투여량(mg)</label>
        <input type="number" id="taurine" value="1000" step="100" min="0">

        <button id="run">시뮬레이션 실행</button>

        <div class="result">
            <h3>결과</h3>
            <p>최종 활성산소 : <span id="ros">-</span></p>
            <p>산화 지표 비율 : <span id="percentage">-</span></p>
        </div>

        <hr>

        <button id="saveCSV">CSV 저장</button>
        <button id="savePNG">그래프 저장</button>
        <button id="reset">초기화</button>

        <hr>

        <div id="analysis">시뮬레이션 결과가 여기에 표시됩니다.</div>
    </div>

    <!-- 오른쪽 영역: 그래프 및 표 -->
    <div class="right">
        <h2>활성산소 변화</h2>
        <canvas id="graph"></canvas>

        <table>
            <tr>
                <th>변수</th>
                <th>최종값</th>
            </tr>
            <tr>
                <td>초과산화물 (Superoxide, R1)</td>
                <td id="r1">-</td>
            </tr>
            <tr>
                <td>과산화수소 (H2O2, R2)</td>
                <td id="r2">-</td>
            </tr>
            <tr>
                <td>환원형 글루타치온 (GSH)</td>
                <td id="gsh">-</td>
            </tr>
            <tr>
                <td>산화형 글루타치온 (GSSG)</td>
                <td id="gssg">-</td>
            </tr>
        </table>
    </div>
</div>

<script>
// Chart.js 초기화
let ctx = document.getElementById("graph");
let graph = new Chart(ctx, {
    type: "line",
    data: {
        labels: [],
        datasets: []
    },
    options: {
        responsive: true,
        animation: false,
        scales: {
            x: { title: { display: true, text: "시간 (Hour)" } },
            y: { title: { display: true, text: "수치 (a.u.)" } }
        }
    }
});

// 효소 상수 정의
const VCAT = 0.8;
const KCAT = 0.6;
const VGPX = 1.2;
const VGR = 1.0;

function generation(I, W, E, S) {
    let X = I + W + E + S;
    return 1 + (5 * X) / (3 + X);
}

function catalase(H2O2, T) {
    let activity = 1 + (0.3 * T) / (1 + T);
    return (VCAT * activity * H2O2) / (KCAT + H2O2);
}

function GPx(H2O2, GSH, T) {
    let activity = 1 + (0.3 * T) / (1 + T);
    let numerator = VGPX * activity * H2O2 * GSH;
    let denominator = 0.25 + 0.5 * GSH + 0.5 * H2O2 + H2O2 * GSH;
    return numerator / (denominator || 1e-6);
}

function GR(GSSG, T) {
    let activity = 1 + (0.3 * T) / (1 + T);
    return (VGR * activity * GSSG) / (0.75 + 1.5 * GSSG);
}

function derivatives(state, input) {
    let R1 = state.R1;
    let R2 = state.R2;
    let G = state.G;
    let Go = state.Go;

    let P = generation(input.I, input.W, input.E, input.S);
    let CAT = catalase(R2, input.T);
    let GPX = GPx(R2, G, input.T);
    let GRR = GR(Go, input.T);

    let dR1 = P - R1 - 0.5 * input.T * R1;
    let dR2 = R1 - CAT - GPX;
    let dG = 2 * GRR - 2 * GPX;
    let dGo = GPX - GRR;

    return { dR1, dR2, dG, dGo };
}

function rk4Step(state, input, dt) {
    let k1 = derivatives(state, input);

    let s2 = {
        R1: Math.max(state.R1 + k1.dR1 * dt / 2, 0),
        R2: Math.max(state.R2 + k1.dR2 * dt / 2, 0),
        G: Math.max(state.G + k1.dG * dt / 2, 0),
        Go: Math.max(state.Go + k1.dGo * dt / 2, 0)
    };
    let k2 = derivatives(s2, input);

    let s3 = {
        R1: Math.max(state.R1 + k2.dR1 * dt / 2, 0),
        R2: Math.max(state.R2 + k2.dR2 * dt / 2, 0),
        G: Math.max(state.G + k2.dG * dt / 2, 0),
        Go: Math.max(state.Go + k2.dGo * dt / 2, 0)
    };
    let k3 = derivatives(s3, input);

    let s4 = {
        R1: Math.max(state.R1 + k3.dR1 * dt, 0),
        R2: Math.max(state.R2 + k3.dR2 * dt, 0),
        G: Math.max(state.G + k3.dG * dt, 0),
        Go: Math.max(state.Go + k3.dGo * dt, 0)
    };
    let k4 = derivatives(s4, input);

    let nextState = {
        R1: Math.max(state.R1 + dt / 6 * (k1.dR1 + 2 * k2.dR1 + 2 * k3.dR1 + k4.dR1), 0),
        R2: Math.max(state.R2 + dt / 6 * (k1.dR2 + 2 * k2.dR2 + 2 * k3.dR2 + k4.dR2), 0),
        G: Math.max(state.G + dt / 6 * (k1.dG + 2 * k2.dG + 2 * k3.dG + k4.dG), 0),
        Go: Math.max(state.Go + dt / 6 * (k1.dGo + 2 * k2.dGo + 2 * k3.dGo + k4.dGo), 0)
    };

    return nextState;
}

function simulate(input) {
    let dt = 0.05;
    let totalTime = 24;
    let steps = totalTime / dt;

    let state = { R1: 1, R2: 0.5, G: 1, Go: 0.05 };

    let time = [];
    let ROS = [];
    let superoxide = [];
    let peroxide = [];
    let GSH = [];
    let GSSG = [];

    for (let i = 0; i <= steps; i++) {
        time.push((i * dt).toFixed(1));
        superoxide.push(state.R1);
        peroxide.push(state.R2);
        GSH.push(state.G);
        GSSG.push(state.Go);
        ROS.push(state.R1 + state.R2);

        state = rk4Step(state, input, dt);
    }

    return { time, ROS, R1: superoxide, R2: peroxide, GSH, GSSG, finalState: state };
}

// 1차 방정식 선형 변환 함수 (R1 값을 2% ~ 7% 비율로 매핑)
function convertToPercentage(R1) {
    let X_min = 0 + 0 + 0 + 0;
    let R1_min = 1 + (5 * X_min) / (3 + X_min); // 1.0

    let X_max = 1 + 1 + 2 + 1;
    let R1_max = 1 + (5 * X_max) / (3 + X_max); // 4.125

    let targetMin = 2; // 2%
    let targetMax = 7; // 7%

    let a = (targetMax - targetMin) / (R1_max - R1_min);
    let b = targetMin - (a * R1_min);

    let percent = (a * R1) + b;
    return percent;
}

function updateGraph(result) {
    graph.data.labels = result.time;
    graph.data.datasets = [
        { label: "총 활성산소 (ROS)", data: result.ROS, borderColor: "#e53935", borderWidth: 3, fill: false },
        { label: "초과산화물", data: result.R1, borderColor: "#fb8c00", borderWidth: 2, fill: false },
        { label: "과산화수소", data: result.R2, borderColor: "#fdd835", borderWidth: 2, fill: false },
        { label: "GSH", data: result.GSH, borderColor: "#43a047", borderWidth: 2, fill: false },
        { label: "GSSG", data: result.GSSG, borderColor: "#8e24aa", borderWidth: 2, fill: false }
    ];
    graph.update();
}

function updateResult(result) {
    let finalR1 = result.finalState.R1;
    let finalROS = result.finalState.R1 + result.finalState.R2;
    let percentage = convertToPercentage(finalR1);

    document.getElementById("ros").innerText = finalROS.toFixed(3);
    document.getElementById("percentage").innerText = percentage.toFixed(2) + " %";
    document.getElementById("r1").innerText = finalR1.toFixed(3);
    document.getElementById("r2").innerText = result.finalState.R2.toFixed(3);
    document.getElementById("gsh").innerText = result.finalState.G.toFixed(3);
    document.getElementById("gssg").innerText = result.finalState.Go.toFixed(3);
}

function analysis(result, input) {
    let ros = result.finalState.R1 + result.finalState.R2;
    let percentage = convertToPercentage(result.finalState.R1);
    
    let txt = "<h3>분석 결과</h3>";
    txt += "<strong>총 활성산소 수준:</strong> " + ros.toFixed(3) + "<br>";
    txt += "<strong>산화 지표 비율:</strong> " + percentage.toFixed(2) + "%<br><br>";

    if (ros < 1) txt += "활성산소가 정상 기준치보다 낮습니다.<br>";
    else if (ros <= 2) txt += "정상 수준의 산화-환원 균형을 유지하고 있습니다.<br>";
    else if (ros <= 4) txt += "면역/염증 반응으로 인해 ROS가 다소 상승한 상태입니다.<br>";
    else txt += "높은 수준의 산화 스트레스 상태입니다. 세포 손상 위험이 있습니다.<br>";

    txt += "<br>설정된 타우린 투여량: <strong>" + (input.T * 1000).toFixed(0) + " mg</strong>";
    document.getElementById("analysis").innerHTML = txt;
}

// 이벤트 리스너
document.getElementById("run").onclick = function() {
    let input = {
        I: Number(document.getElementById("infection").value),
        W: Number(document.getElementById("wound").value),
        E: Number(document.getElementById("exercise").value),
        S: Number(document.getElementById("sleep").value),
        T: Number(document.getElementById("taurine").value) / 1000
    };

    let result = simulate(input);
    updateGraph(result);
    updateResult(result);
    analysis(result, input);
};

document.getElementById("saveCSV").onclick = function() {
    if (!graph.data.labels.length) return alert("먼저 시뮬레이션을 실행하세요.");
    let csv = "시간(Hour),총_ROS,초과산화물,과산화수소,GSH,GSSG\n";
    for (let i = 0; i < graph.data.labels.length; i++) {
        csv += `${graph.data.labels[i]},${graph.data.datasets[0].data[i]},${graph.data.datasets[1].data[i]},${graph.data.datasets[2].data[i]},${graph.data.datasets[3].data[i]},${graph.data.datasets[4].data[i]}\n`;
    }
    let blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
    let url = URL.createObjectURL(blob);
    let a = document.createElement("a");
    a.href = url;
    a.download = "ROS_simulation_data.csv";
    a.click();
};

document.getElementById("savePNG").onclick = function() {
    if (!graph.data.labels.length) return alert("먼저 시뮬레이션을 실행하세요.");
    let a = document.createElement("a");
    a.href = graph.toBase64Image();
    a.download = "ROS_graph.png";
    a.click();
};

document.getElementById("reset").onclick = function() {
    document.getElementById("infection").value = 0;
    document.getElementById("wound").value = 0;
    document.getElementById("exercise").value = 0;
    document.getElementById("sleep").value = 0;
    document.getElementById("taurine").value = 1000;

    graph.data.labels = [];
    graph.data.datasets = [];
    graph.update();

    document.getElementById("ros").innerText = "-";
    document.getElementById("percentage").innerText = "-";
    document.getElementById("r1").innerText = "-";
    document.getElementById("r2").innerText = "-";
    document.getElementById("gsh").innerText = "-";
    document.getElementById("gssg").innerText = "-";
    document.getElementById("analysis").innerHTML = "시뮬레이션 결과가 여기에 표시됩니다.";
};
</script>

</body>
</html>
