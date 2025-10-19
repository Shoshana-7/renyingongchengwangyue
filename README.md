<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>王岳的网页 - 瓶颈工序分析</title>
    <style>
        .chat-container {
            width: 600px;
            margin: 50px auto;
            border: 1px solid #ccc;
            border-radius: 5px;
            overflow: hidden;
        }
        .chat-messages {
            height: 400px;
            padding: 10px;
            overflow-y: auto;
        }
        .message {
            margin-bottom: 10px;
            padding: 8px;
            border-radius: 5px;
            max-width: 80%;
            word-wrap: break-word;
        }
        .user {
            background-color: #e6f7ff;
            text-align: right;
            margin-left: auto;
        }
        .assistant {
            background-color: #f5f5f5;
            text-align: left;
            margin-right: auto;
        }
        .chat-input {
            display: flex;
            border-top: 1px solid #ccc;
        }
        #input {
            flex: 1;
            padding: 10px;
            border: none;
            outline: none;
            border-right: 1px solid #ccc;
        }
        #send {
            padding: 0 20px;
            background-color: #007bff;
            color: #fff;
            border: none;
            cursor: pointer;
        }
        #send:hover {
            background-color: #0056b3;
        }
        .tip {
            font-size: 12px;
            color: #666;
            padding: 5px 10px;
            background-color: #fafafa;
            border-bottom: 1px solid #eee;
        }
        .process-list {
            font-size: 12px;
            color: #333;
            margin-top: 5px;
            padding-top: 5px;
            border-top: 1px dashed #eee;
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="tip">
            请逐条输入工序信息（每条工序单独发送），格式：<br>
            工序名称,工序时间(秒),前置工序1,前置工序2,...（无前置工序则留空，用英文逗号分隔）<br>
            示例1（无前置）：注塑灯座,300<br>
            示例2（有前置）：焊接线路,480,注塑灯座<br>
            所有工序输入完成后，发送“开始分析”即可。
        </div>
        <div class="chat-messages" id="messages"></div>
        <div class="chat-input">
            <input type="text" id="input" placeholder="按格式输入单条工序（时间单位：秒），完成后发送“开始分析”...">
            <button id="send">发送</button>
        </div>
    </div>

    <script>
        const messagesContainer = document.getElementById('messages');
        const input = document.getElementById('input');
        const sendBtn = document.getElementById('send');
        let processes = {}; // 存储所有工序：{名称: [时间(秒), [前置工序1, 前置工序2...]]}

        // 显示消息
        function showMessage(content, isUser = false) {
            const message = document.createElement('div');
            message.className = `message ${isUser ? 'user' : 'assistant'}`;
            message.innerHTML = content.replace(/\n/g, '<br>');
            messagesContainer.appendChild(message);
            messagesContainer.scrollTop = messagesContainer.scrollHeight;
        }

        // 显示已添加的工序列表（辅助用户确认）
        function showProcessList() {
            if (Object.keys(processes).length === 0) return '';
            let list = '<div class="process-list">已添加工序：<br>';
            Object.entries(processes).forEach(([name, [time, pres]]) => {
                list += `- ${name}（耗时${time}秒，前置：${pres.length > 0 ? pres.join('、') : '无'}）<br>`;
            });
            list += '</div>';
            return list;
        }

        // ---------------------- 核心算法 ----------------------
        function checkCircularDependency(processes) {
            const visited = new Set();
            const recStack = new Set();

            function dfs(process) {
                if (recStack.has(process)) return true;
                if (visited.has(process)) return false;
                visited.add(process);
                recStack.add(process);
                const [_, predecessors] = processes[process] || [0, []];
                for (const prev of predecessors) {
                    if (dfs(prev)) return true;
                }
                recStack.delete(process);
                return false;
            }

            for (const process of Object.keys(processes)) {
                if (dfs(process)) return true;
            }
            return false;
        }

        function findBottleneck(processes) {
            const sortedProcesses = [];
            const visited = new Set();

            function dfs(process) {
                if (visited.has(process)) return;
                const [_, predecessors] = processes[process] || [0, []];
                for (const prev of predecessors) {
                    dfs(prev);
                }
                visited.add(process);
                sortedProcesses.push(process);
            }

            for (const process of Object.keys(processes)) {
                dfs(process);
            }

            const es = {};
            const ef = {};
            Object.keys(processes).forEach(p => es[p] = 0);

            for (const p of sortedProcesses) {
                const [duration, predecessors] = processes[p];
                if (predecessors.length === 0) {
                    ef[p] = duration;
                } else {
                    es[p] = Math.max(...predecessors.map(prev => ef[prev]));
                    ef[p] = es[p] + duration;
                }
            }

            const totalTime = Math.max(...Object.values(ef));
            const lf = {};
            const ls = {};
            Object.keys(processes).forEach(p => lf[p] = totalTime);

            for (const p of sortedProcesses.reverse()) {
                const [duration, _] = processes[p];
                const successors = Object.keys(processes).filter(s => processes[s][1].includes(p));

                if (successors.length === 0) {
                    ls[p] = lf[p] - duration;
                } else {
                    lf[p] = Math.min(...successors.map(s => ls[s]));
                    ls[p] = lf[p] - duration;
                }
            }

            const criticalPath = Object.keys(processes).filter(p => es[p] === ls[p]);
            if (criticalPath.length === 0) return null;

            const bottleneck = criticalPath.reduce((maxP, p) => {
                return processes[p][0] > processes[maxP][0] ? p : maxP;
            }, criticalPath[0]);

            return {
                totalTime: totalTime.toFixed(1),
                criticalPath: criticalPath.join(' → '),
                bottleneck: bottleneck,
                bottleneckTime: processes[bottleneck][0].toFixed(1)
            };
        }
        // ------------------------------------------------------

        // 处理发送事件
        function handleSend() {
            const content = input.value.trim();
            if (!content) return;

            // 显示用户输入
            showMessage(content, true);
            input.value = '';

            // 判断是否触发分析
            if (content === '开始分析') {
                if (Object.keys(processes).length === 0) {
                    showMessage('还没有添加任何工序哦～ 请先按格式输入工序。');
                    return;
                }

                // 检查循环依赖
                if (checkCircularDependency(processes)) {
                    showMessage('检测到工序存在循环依赖（例如A依赖B，B又依赖A），请修改工序关系后重试。');
                    return;
                }

                // 执行分析
                const result = findBottleneck(processes);
                if (result) {
                    showMessage(`
                    🔍 瓶颈工序分析结果：<br>
                    1. 总生产时间：${result.totalTime} 秒<br>
                    2. 关键路径（最长生产路径）：${result.criticalPath}<br>
                    3. 瓶颈工序：${result.bottleneck}<br>
                    4. 瓶颈耗时：${result.bottleneckTime} 秒<br>
                    💡 建议：优先优化瓶颈工序以提升整体产能。
                    `);
                    // 分析后清空，方便重新输入
                    processes = {};
                } else {
                    showMessage('分析失败，请检查工序输入是否正确。');
                }
                return;
            }

            // 处理单条工序输入（格式：名称,时间(秒),前置1,前置2,...）
            const parts = content.split(',').map(item => item.trim());
            if (parts.length < 2) {
                showMessage('格式错误！正确格式：工序名称,工序时间(秒),前置工序1,前置工序2...（最少需要名称和时间，用英文逗号分隔）');
                return;
            }

            const name = parts[0];
            const time = parseFloat(parts[1]);
            const predecessors = [];

            // 提取前置工序（过滤空值）
            for (let i = 2; i < parts.length; i++) {
                if (parts[i]) { // 非空字符串才视为前置工序
                    predecessors.push(parts[i]);
                }
            }

            // 验证工序名称
            if (!name) {
                showMessage('工序名称不能为空，请重新输入。');
                return;
            }

            // 验证时间（秒级，必须为正数）
            if (isNaN(time) || time <= 0) {
                showMessage(`“${name}”的时间必须是正数（单位：秒），请重新输入。`);
                return;
            }

            // 验证前置工序是否存在
            const invalidPres = predecessors.filter(p => !processes[p]);
            if (invalidPres.length > 0) {
                showMessage(`“${name}”的前置工序${invalidPres.join('、')}尚未添加，请先添加这些工序。`);
                return;
            }

            // 保存工序
            processes[name] = [time, predecessors];
            showMessage(`✅ 已添加工序：<br>
            名称：${name}<br>
            时间：${time} 秒<br>
            前置工序：${predecessors.length > 0 ? predecessors.join('、') : '无'}<br>
            ${showProcessList()}<br>
            可继续添加其他工序，完成后发送“开始分析”。`);
        }

        // 绑定事件
        sendBtn.addEventListener('click', handleSend);
        input.addEventListener('keydown', (e) => {
            if (e.key === 'Enter') handleSend();
        });

        // 初始化欢迎消息
        window.onload = () => {
            showMessage('你好！请逐条输入生产工序，每条工序格式为：<br>工序名称,工序时间(秒),前置工序1,前置工序2...<br>（无前置工序可留空，例如“注塑灯座,300”）<br>所有工序输入完成后，发送“开始分析”即可得到结果。');
        };
    </script>
</body>
</html>
