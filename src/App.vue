<script setup>
import { ref, onMounted, watch, computed } from "vue";

// --- 1. 状态定义 ---
const currentInput = ref(""); // 当前输入（下方大字）
const lastExpression = ref(""); // 上一次的表达式（上方小字）
const history = ref(
  JSON.parse(localStorage.getItem("calc_history") || "[]").slice(0, 10),
);
const isDark = ref(localStorage.getItem("theme") === "dark");
const activeTab = ref("calc");

// --- 2. 计算器核心逻辑 ---
const append = (char) => (currentInput.value += char);
const clearAll = () => {
  currentInput.value = "";
  lastExpression.value = "";
};
const backspace = () => (currentInput.value = currentInput.value.slice(0, -1));
const appendSci = (func) => (currentInput.value += func + "(");

const clearHistory = () => {
  if (history.value.length > 0 && confirm("确定要清空所有历史记录吗？")) {
    history.value = [];
  }
};

const calculate = () => {
  try {
    if (!currentInput.value) return;

    // 备份原始表达式用于显示
    const originalExpr = currentInput.value;

    let expr = currentInput.value
      .replace(/sin\(/g, "Math.sin(")
      .replace(/sqrt\(/g, "Math.sqrt(")
      .replace(/pow2\(/g, "Math.pow(");

    if (expr.includes("Math.pow(")) {
      if (!expr.includes(",")) {
        expr = expr.replace(/Math\.pow\(([^)]+)/g, "Math.pow($1, 2");
      }
    }

    const leftCount = (expr.match(/\(/g) || []).length;
    const rightCount = (expr.match(/\)/g) || []).length;
    for (let i = 0; i < leftCount - rightCount; i++) expr += ")";

    if (expr.includes("/0")) throw new Error("除数不能为0");

    const res = eval(expr);
    if (res === Infinity || isNaN(res)) throw new Error("错误");

    const formatted = parseFloat(res.toFixed(4)).toString();

    // 更新显示状态
    lastExpression.value = originalExpr + " =";
    currentInput.value = formatted;

    // 存入历史
    history.value.unshift(`${originalExpr} = ${formatted}`);
    if (history.value.length > 10) history.value = history.value.slice(0, 10);
  } catch (e) {
    currentInput.value = "Error";
    lastExpression.value = "";
  }
};

const useHistory = (rec) => {
  const parts = rec.split(" = ");
  lastExpression.value = parts[0] + " =";
  currentInput.value = parts[1];
};

// --- 3. 持久化与主题 ---
watch(history, (v) => localStorage.setItem("calc_history", JSON.stringify(v)), {
  deep: true,
});
watch(
  isDark,
  (val) => {
    localStorage.setItem("theme", val ? "dark" : "light");
    document.documentElement.setAttribute("data-theme", val ? "dark" : "light");
  },
  { immediate: true },
);

const toggleTheme = () => (isDark.value = !isDark.value);

// --- 4. 键盘监听 ---
onMounted(() => {
  window.addEventListener("keydown", (e) => {
    if (activeTab.value !== "calc") return;
    if (/[0-9\+\-\*\/\.\%\(\)]/.test(e.key)) append(e.key);
    if (e.key === "Enter") {
      e.preventDefault();
      calculate();
    }
    if (e.key === "Backspace") backspace();
    if (e.key === "Escape") clearAll();
  });
});

// --- 5. 单位转换 ---
const unitVal = ref(0);
const unitType = ref("length");
const fromUnit = ref("m");
const toUnit = ref("km");

const rates = {
  length: { m: 1, km: 0.001, cm: 100, mm: 1000 },
  weight: { kg: 1, g: 1000, mg: 1000000, t: 0.001 },
  temp: { C: 1, F: 1, K: 1 },
};

const convertedVal = computed(() => {
  if (unitType.value === "temp") {
    let c =
      fromUnit.value === "F"
        ? ((unitVal.value - 32) * 5) / 9
        : fromUnit.value === "K"
          ? unitVal.value - 273.15
          : unitVal.value;
    if (toUnit.value === "F") return ((c * 9) / 5 + 32).toFixed(2);
    if (toUnit.value === "K") return (c + 273.15).toFixed(2);
    return c.toFixed(2);
  }
  return (
    (unitVal.value / rates[unitType.value][fromUnit.value]) *
    rates[unitType.value][toUnit.value]
  ).toFixed(4);
});
</script>

<template>
  <div class="app-shell" :class="{ dark: isDark }">
    <div class="main-container">
      <div class="nav-header">
        <button
          @click="activeTab = 'calc'"
          :class="{ active: activeTab === 'calc' }"
        >
          计算器
        </button>
        <button
          @click="activeTab = 'unit'"
          :class="{ active: activeTab === 'unit' }"
        >
          单位转换
        </button>
        <button @click="toggleTheme" class="theme-toggle">
          {{ isDark ? "🌞" : "🌙" }}
        </button>
      </div>

      <div v-if="activeTab === 'calc'" class="fade-in">
        <div class="display">
          <div class="exp-line">{{ lastExpression }}</div>
          <div class="val-line">
            <input v-model="currentInput" placeholder="0" readonly />
          </div>
        </div>

        <div class="keypad">
          <button @click="clearAll" class="btn-danger">AC</button>
          <button @click="backspace" class="btn-warn">←</button>
          <button @click="append('%')">%</button>
          <button @click="appendSci('sqrt')" class="btn-sci">√</button>
          <button @click="append('/')" class="btn-op">÷</button>

          <button @click="append('7')">7</button>
          <button @click="append('8')">8</button>
          <button @click="append('9')">9</button>
          <button @click="appendSci('sin')" class="btn-sci">sin</button>
          <button @click="append('*')" class="btn-op">×</button>

          <button @click="append('4')">4</button>
          <button @click="append('5')">5</button>
          <button @click="append('6')">6</button>
          <button @click="appendSci('pow2')" class="btn-sci">x²</button>
          <button @click="append('-')" class="btn-op">-</button>

          <button @click="append('1')">1</button>
          <button @click="append('2')">2</button>
          <button @click="append('3')">3</button>
          <button @click="append('(')">(</button>
          <button @click="append('+')" class="btn-op">+</button>

          <button @click="append('0')" class="span-2">0</button>
          <button @click="append('.')">.</button>
          <button @click="append(')')">)</button>
          <button @click="calculate" class="btn-equal">=</button>
        </div>

        <div class="history-panel">
          <div class="h-header-row">
            <span class="h-head">历史记录</span>
            <button
              @click="clearHistory"
              class="btn-clear-history"
              v-if="history.length > 0"
            >
              清空
            </button>
          </div>

          <div v-if="history.length > 0" class="history-list">
            <ul>
              <li v-for="(h, i) in history" :key="i" @click="useHistory(h)">
                {{ h }}
              </li>
            </ul>
          </div>
          <div v-else class="empty-state">
            <div class="empty-icon">📂</div>
            <p>暂无历史计算记录</p>
          </div>
        </div>
      </div>

      <div v-else class="unit-box fade-in">
        <select
          v-model="unitType"
          @change="
            fromUnit = Object.keys(rates[unitType])[0];
            toUnit = Object.keys(rates[unitType])[1];
          "
        >
          <option value="length">长度转换</option>
          <option value="weight">重量转换</option>
          <option value="temp">温度转换</option>
        </select>
        <div class="convert-row">
          <input type="number" v-model="unitVal" />
          <select v-model="fromUnit">
            <option
              v-for="u in Object.keys(rates[unitType])"
              :key="u"
              :value="u"
            >
              {{ u }}
            </option>
          </select>
        </div>
        <div class="arrow">↓</div>
        <div class="convert-row">
          <input :value="convertedVal" readonly />
          <select v-model="toUnit">
            <option
              v-for="u in Object.keys(rates[unitType])"
              :key="u"
              :value="u"
            >
              {{ u }}
            </option>
          </select>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --bg: #f0f2f5;
  --card: #ffffff;
  --text: #333;
  --btn: #f1f3f5;
  --hover: #e2e6ea;
  --exp: #888;
}
[data-theme="dark"] {
  --bg: #121212;
  --card: #1e1e1e;
  --text: #e0e0e0;
  --btn: #333333;
  --hover: #444444;
  --exp: #aaa;
}

.app-shell {
  background: var(--bg);
  color: var(--text);
  min-height: 100vh;
  transition: 0.3s;
  padding: 20px;
  font-family:
    -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}
.main-container {
  max-width: 400px;
  margin: 0 auto;
  background: var(--card);
  padding: 20px;
  border-radius: 24px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.1);
}

/* 导航 */
.nav-header {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}
.nav-header button {
  flex: 1;
  padding: 10px;
  border: none;
  background: var(--btn);
  color: var(--text);
  cursor: pointer;
  border-radius: 12px;
  font-size: 14px;
  transition: 0.2s;
}
.nav-header button.active {
  background: #42b983;
  color: white;
}
.theme-toggle {
  flex: 0.3 !important;
  font-size: 18px;
}

/* 双行显示屏 */
.display {
  background: var(--bg);
  border-radius: 16px;
  padding: 15px;
  margin-bottom: 20px;
  min-height: 100px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.exp-line {
  height: 24px;
  font-size: 14px;
  color: var(--exp);
  text-align: right;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-bottom: 5px;
}
.val-line input {
  width: 100%;
  border: none;
  background: transparent;
  font-size: 32px;
  text-align: right;
  color: var(--text);
  font-weight: bold;
  padding: 0;
  outline: none;
}

/* 按键区 */
.keypad {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 10px;
}
button {
  height: 50px;
  border: none;
  border-radius: 14px;
  background: var(--btn);
  color: var(--text);
  cursor: pointer;
  transition: all 0.2s;
  font-weight: bold;
  font-size: 16px;
}
button:hover {
  background: var(--hover);
  transform: translateY(-2px);
}
button:active {
  transform: translateY(0);
}

.btn-op {
  background: #ff9f0a;
  color: white;
}
.btn-sci {
  background: #e7f5ff;
  color: #228be6;
}
[data-theme="dark"] .btn-sci {
  background: #1a365d;
  color: #63b3ed;
}
.btn-equal {
  background: #42b983;
  color: white;
}
.span-2 {
  grid-column: span 2;
}
.btn-danger {
  background: #ff4d4f;
  color: white;
}
.btn-warn {
  background: #faad14;
  color: white;
}

/* 历史记录 & 空状态 */
.history-panel {
  margin-top: 25px;
  border-top: 2px solid var(--btn);
  padding-top: 15px;
}
.h-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}
.h-head {
  font-size: 13px;
  font-weight: bold;
  color: var(--exp);
}
.btn-clear-history {
  height: 24px;
  font-size: 11px;
  padding: 0 10px;
  background: #ff4d4f;
  color: white;
  border-radius: 6px;
}
.history-list {
  max-height: 120px;
  overflow-y: auto;
}
li {
  padding: 10px;
  font-size: 13px;
  cursor: pointer;
  border-radius: 10px;
  margin-bottom: 5px;
  background: var(--btn);
  opacity: 0.8;
  transition: 0.2s;
}
li:hover {
  opacity: 1;
  background: var(--hover);
}

.empty-state {
  text-align: center;
  padding: 20px;
  color: var(--exp);
}
.empty-icon {
  font-size: 32px;
  margin-bottom: 8px;
  opacity: 0.5;
}
.empty-state p {
  font-size: 12px;
  margin: 0;
}

/* 单位转换 */
.unit-box select,
.unit-box input {
  width: 100%;
  padding: 12px;
  margin: 10px 0;
  border-radius: 12px;
  border: 1px solid var(--btn);
  background: var(--bg);
  color: var(--text);
  box-sizing: border-box;
  font-size: 16px;
}
.convert-row {
  display: flex;
  gap: 10px;
}
.arrow {
  text-align: center;
  font-size: 20px;
  margin: 5px 0;
  opacity: 0.6;
}

.fade-in {
  animation: fadeIn 0.4s ease-out;
}
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
