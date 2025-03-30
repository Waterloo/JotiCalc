<script setup lang="ts">
import { ref, reactive, onMounted, nextTick, watch } from 'vue';
import * as math from 'mathjs';
import tacklet from 'tacklet'
// Types
interface CalculatorLine {
  input: string;
  result: string;
  hasError: boolean;
  errorMessage: string;
}

// References
const tooltip = ref<HTMLElement | null>(null);
const lineInputs = ref<HTMLInputElement[]>([]);
const showHelpPanel = ref(false);

// State
const variables = reactive<Record<string, any>>({});
const lines = ref<CalculatorLine[]>([]);

// Initial expressions
const initialExpressions = [
  'users = 500',
  'dataPerUser = 100 KB',
  'totalData = users * dataPerUser',
  'time = 2 hr + 30 minutes',
  'distance = 5 km + 200 m',
  'temperature = 72 degF to degC',
  '// Try your own calculations below',
  ''
];

// Define custom units
function defineCustomUnits() {
  try {
    // Data storage units (binary/1024-based)
    math.createUnit('KB', '1024 bytes');
    math.createUnit('MB', '1024 KB');
    math.createUnit('GB', '1024 MB');
    math.createUnit('TB', '1024 GB');
    math.createUnit('PB', '1024 TB');
    
    // Currency units - base USD
    math.createUnit('USD', { baseName: 'currency' });
    
    // Internet/network speed units
    math.createUnit('Kbps', '1000 bit/s');
    math.createUnit('Mbps', '1000 Kbps');
    math.createUnit('Gbps', '1000 Mbps');
    
    // Digital photography/screen units
    math.createUnit('MP', '1000000');  // Megapixel for camera resolutions
    math.createUnit('DPI', '1 inch^-1');  // Dots per inch for printing
    math.createUnit('PPI', '1 inch^-1');  // Pixels per inch for screens
    
    // Social media units
    math.createUnit('like', '1');
    math.createUnit('view', '1');
    math.createUnit('impression', '1');
    math.createUnit('engagement', '1');
    
    // Software development units
    math.createUnit('LOC', '1');  // Lines of code
    math.createUnit('FP', '1');   // Function points
    math.createUnit('PR', '1');   // Pull request
    math.createUnit('issue', '1');
    math.createUnit('bug', '1');
    
    console.log("Custom units defined successfully");
  } catch (e) {
    console.error("Error defining custom units:", e);
  }
}

async function fetchCurrency(){
  try{
    const response = await fetch('https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/usd.json');
    if (!response.ok) {
      throw new Error('Failed to fetch currency data');
    }
    const data = await response.json();

    if (!data.usd) {
      throw new Error('Invalid currency data format');
    }

    // Create currency units
    for (const [currency, rate] of Object.entries(data.usd)) {
      // Skip invalid currencies and special cases
      if (
        currency === 'usd' || 
        currency === '1inch' || 
        typeof rate !== 'number' || 
        !isFinite(rate) || 
        rate <= 0
      ) {
        continue;
      }

      try {
        // Create lowercase unit for conversion to USD
        math.createUnit(currency.toLowerCase(), `${1 / rate} USD`);
        // Create uppercase unit for conversion from USD
        math.createUnit(currency.toUpperCase(), `${1 / rate} USD`);
      } catch (err) {
        console.warn(`Failed to create currency unit ${currency}:`, err);
        }
      }
  } catch (err) {
    console.error('Error fetching currency data:', err);
  }
}
  
// Evaluate a single line
function evaluateLine(index: number) {
  const line = lines.value[index];
  const input = line?.input?.trim();
  // Skip empty lines or comments
  if (!input || input.startsWith('//')) {
    line.result = '';
    line.hasError = false;
    line.errorMessage = '';
    return;
  }
  
  try {
    // Handle variable assignment
    if (input.includes('=')) {
      const [varName, expression] = input.split('=').map(part => part.trim());
      
      // Check for unit conversion
      if (expression.includes(' to ')) {
        const [fromExpr, toUnit] = expression.split(' to ').map(part => part.trim());
        const valueWithUnit = math.evaluate(fromExpr, variables);
        const converted = math.to(valueWithUnit, toUnit);
        variables[varName] = converted;
        line.result = math.format(converted, { precision: 14 });
      } else {
        // Regular evaluation with units
        const value = math.evaluate(expression, variables);
        variables[varName] = value;
        line.result = math.format(value, { precision: 14 });
      }
    } else {
      // Just an expression to evaluate
      const value = math.evaluate(input, variables);
      line.result = math.format(value, { precision: 14 });
    }
    
    line.hasError = false;
    line.errorMessage = '';
  } catch (err: any) {
    line.result = 'Error';
    line.hasError = true;
    line.errorMessage = err.message || 'Unknown error';
  }
  
  // Update all lines after this one
  updateAllLinesAfter(index);
}

// Update all lines after a specific index
function updateAllLinesAfter(startIndex: number) {
  for (let i = startIndex + 1; i < lines.value.length; i++) {
    evaluateLine(i);
  }
}

// Add a new line
function addLine(index?: number) {
  const newLine = {
    input: '',
    result: '',
    hasError: false,
    errorMessage: ''
  };
  
  if (typeof index === 'number') {
    lines.value.splice(index, 0, newLine);
  } else {
    lines.value.push(newLine);
  }
  
  // Focus the new input after the DOM has updated
  nextTick(() => {
    const inputs = lineInputs.value;
    const focusIndex = typeof index === 'number' ? index : inputs.length - 1;
    if (inputs && inputs[focusIndex]) {
      inputs[focusIndex].focus();
    }
  });
}

// Handle backspace key on empty lines
function handleBackspace(event: KeyboardEvent, index: number) {
  const line = lines.value[index];
  
  if (line.input === '' && lines.value.length > 1) {
    event.preventDefault();
    
    // Remove the line
    lines.value.splice(index, 1);
    
    // Focus the previous line if possible
    nextTick(() => {
      const inputs = lineInputs.value;
      const focusIndex = Math.min(index, inputs.length - 1);
      if (focusIndex >= 0) {
        inputs[focusIndex].focus();
      }
    });
  }
}

// Toggle help panel
const helpPanel = ref<HTMLElement | null>(null);
function toggleHelp() {
  showHelpPanel.value = !showHelpPanel.value;
  location.hash = '';
  location.hash = '#help'
}

// Show tooltip
function showTooltip(event: MouseEvent, message: string) {
  console.log(message, "hover", message)
  if (!tooltip.value || !message) return;
  
  const target = event.target as HTMLElement;
  const rect = target.getBoundingClientRect();
  
  tooltip.value.textContent = message;
  tooltip.value.style.top = `${rect.bottom + 10}px`;
  tooltip.value.style.left = `${rect.left + rect.width / 2 - 100}px`;
  tooltip.value.style.opacity = '1';
}

// Hide tooltip
function hideTooltip() {
  if (tooltip.value) {
    tooltip.value.style.opacity = '0';
  }
}

// Initialize the component
onMounted(async () => {
  defineCustomUnits();
  
  // Evaluate initial lines
  fetchCurrency();
  await tacklet.connect();
  console.log('Connected to tacklet');
  const data = await tacklet.getData();
  lines.value = data.lines || initialExpressions.map(expr => {
    return ({
      input: expr,
      result: '',
      hasError: false,
      errorMessage: ''
    });
  });
  console.log(lines.value)
  lines.value.forEach((_, index) => {
    evaluateLine(index);
  });
});

watch(lines, () => {
  tacklet.setData({lines: JSON.parse(JSON.stringify(lines.value))});
  console.log('setting data')
}, {deep: true})


</script>

<template>
   <div class="max-w-5xl mx-auto">
    <!-- Header -->
    <div class="bg-gradient-to-r from-primary to-primary-dark  p-6 md:p-10 text-white relative overflow-hidden">
      <div class="relative z-10">
        <h1 class="text-3xl font-bold mb-2 text-">JotiCalc</h1>
        <p class="text-lg opacity-90 max-w-xl">Advanced calculator with support for variables, units, and conversions. Type expressions and see results instantly.</p>
      </div>
      <!-- Decorative background element -->
      <div class="absolute top-[-50%] right-[-10%] w-3/5 h-[200%] bg-white bg-opacity-5 transform rotate-12 rounded-full pointer-events-none"></div>
    </div>
    
    <div class="bg-white rounded-b-lg shadow-lg overflow-hidden mb-8">
      <!-- Action buttons -->
      <div class="flex p-4 gap-4 items-center bg-gray-50 border-b border-gray-200">
        <button @click="addLine" class="flex items-center gap-2 bg-primary hover:bg-primary-dark text-white font-medium py-2 px-4 rounded transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <line x1="12" y1="5" x2="12" y2="19"></line>
            <line x1="5" y1="12" x2="19" y2="12"></line>
          </svg>
          Add Line
        </button>
        <button @click="toggleHelp" class="flex items-center gap-2 border border-gray-300 text-gray-600 hover:bg-gray-100 hover:text-gray-800 font-medium py-2 px-4 rounded transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"></path>
            <line x1="12" y1="17" x2="12.01" y2="17"></line>
          </svg>
          Help & Examples
        </button>
        <div class="flex-grow"></div>
        <div class="text-gray-500 text-sm">Press Enter to add a new line</div>
      </div>
      <div ref="helpPanel" id="help" :class="{ 'max-h-0': !showHelpPanel, 'max-h-[800px]': showHelpPanel }" class="overflow-hidden transition-all duration-300 ease-in-out bg-gray-50 border-t border-gray-200">
        <div class="p-6 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-8">
          <!-- Basic Usage -->
          <div>
            <h3 class="text-lg font-semibold text-gray-800 mb-3 flex items-center gap-2">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-primary" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <polyline points="4 17 10 11 4 5"></polyline>
                <line x1="12" y1="19" x2="20" y2="19"></line>
              </svg>
              Basic Usage
            </h3>
            <ul class="space-y-1.5 text-gray-700">
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Type mathematical expressions
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Assign variables with <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">name = value</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Use variables in subsequent calculations
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Start lines with <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">//</span> for comments
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Press Enter to add new lines
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Use units directly: <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">5 km + 200 m</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                Convert units: <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">72 degF to degC</span>
              </li>
            </ul>
          </div>
          
          <!-- Physical Units -->
          <div>
            <h3 class="text-lg font-semibold text-gray-800 mb-3 flex items-center gap-2">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-primary" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <circle cx="12" cy="12" r="10"></circle>
                <line x1="8" y1="12" x2="16" y2="12"></line>
                <line x1="12" y1="8" x2="12" y2="16"></line>
              </svg>
              Physical Units
            </h3>
            <ul class="space-y-1.5 text-gray-700">
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Length:</strong> mm, cm, m, km, inch, ft, mi
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Mass:</strong> g, kg, lb, oz, ton
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Time:</strong> s, min, hour, day, week
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Temperature:</strong> degC, degF, K
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Energy:</strong> J, kWh, cal, BTU
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Force:</strong> N, lbf, kgf
              </li>
            </ul>
          </div>
          
          <!-- Custom Units -->
          <div>
            <h3 class="text-lg font-semibold text-gray-800 mb-3 flex items-center gap-2">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-primary" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <polyline points="22 12 16 12 14 15 10 15 8 12 2 12"></polyline>
                <path d="M5.45 5.11L2 12v6a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-6l-3.45-6.89A2 2 0 0 0 16.76 4H7.24a2 2 0 0 0-1.79 1.11z"></path>
              </svg>
              Custom Units
            </h3>
            <ul class="space-y-1.5 text-gray-700">
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Data:</strong> bit, byte, KB, MB, GB, TB, PB
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Currency:</strong> USD, EUR, GBP, JPY, CAD
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Network:</strong> Kbps, Mbps, Gbps
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Digital:</strong> MP, DPI, PPI
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Social:</strong> like, view, impression
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <strong>Dev:</strong> LOC, FP, PR, issue, bug
              </li>
            </ul>
          </div>
          
          <!-- Examples -->
          <div>
            <h3 class="text-lg font-semibold text-gray-800 mb-3 flex items-center gap-2">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-primary" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                <line x1="9" y1="9" x2="15" y2="15"></line>
                <line x1="15" y1="9" x2="9" y2="15"></line>
              </svg>
              Examples
            </h3>
            <ul class="space-y-2 text-gray-700">
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">users = 500</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">dataPerUser = 100 KB</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">time = 2 hr + 30 min</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">temp = 98.6 degF to degC</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">budget = 1000 USD to EUR</span>
              </li>
              <li class="flex items-center">
                <span class="text-primary font-bold mr-2">•</span>
                <span class="font-mono text-sm text-primary-dark bg-gray-100 px-1.5 rounded">speed = 60 km / 1 hr</span>
              </li>
            </ul>
          </div>
        </div>
      </div>

      <!-- Editor container -->
      <div class="flex flex-col h-[450px] border-b border-gray-200">
        <div class="flex-grow overflow-y-auto py-4 border-t border-gray-200">
          <div v-for="(line, index) in lines" :key="index" class="flex items-center px-4 py-1 group gap-3 text-left">
            <div class="font-mono text-gray-500">{{index + 1}}</div>
            <input
              v-model="line.input"
              ref="lineInputs"
              type="text"
              class="flex-grow bg-transparent border-none focus:ring-0 font-mono text-gray-800"
              :class="{ 'text-red-500': line.hasError }"
              @input="evaluateLine(index)"
              @keydown.enter.prevent="addLine(index+1)"
              @keydown.backspace="handleBackspace($event, index)"
            >
            <div class="w-48 text-right font-mono" :class="[line.hasError ? 'text-red-500' : 'text-green-600']"  @mouseenter="showTooltip($event, line.errorMessage)"
            @mouseleave="hideTooltip">{{ line.result }}</div>
          </div>
        </div>
      </div>
      
      <!-- Help panel (initially hidden) -->
    </div>
  </div>
  
  <!-- Tooltip for error messages -->
  <div ref="tooltip" class="fixed bg-gray-800 text-white py-2 px-3 rounded text-sm shadow-md pointer-events-none opacity-0 transition-opacity duration-200 z-50"></div>

</template>