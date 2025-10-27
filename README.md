# SIMPLE-SCOPE
裸机开发：支持抢占的前后台增强架构（示波器场景实践）
裸机开发中完全可以实现 “不浪费资源又能抢占” 的软件架构，核心是通过 “中断优先级嵌套 + 任务状态管理 + 非阻塞式任务拆分” 实现，无需 RTOS 也能达到接近 RTOS 的抢占效果，且资源开销极小（仅需几个全局变量和状态标记）。这种架构被称为 “前后台架构增强版”，兼顾了裸机的轻量性和抢占式的实时性。
一、核心设计思路
前台（中断层）
用 NVIC 中断优先级控制 “抢占权”，高优先级中断可打断低优先级中断和后台任务，仅负责 “紧急事件触发”（如用户按键、故障报警），不执行复杂逻辑。
后台（主循环）
执行非紧急任务（如波形刷新、数据处理），但任务需拆分成 “小步” 执行，每步结束后检查 “是否有高优先级任务请求”，主动让出 CPU。
状态管理
用全局变量标记任务的 “运行状态”“暂停点” 和 “中间数据”，确保抢占后能从断点恢复，避免数据混乱或进度丢失。
二、具体实现（示波器场景）
以 “FFT 分析（高优先级，可抢占）” 和 “波形刷新（低优先级，可被抢占）” 为例，完整实现如下：
2.1 任务拆分与状态定义
将低优先级任务（波形刷新）拆分为非阻塞小步骤，用枚举记录任务状态，全局变量保存中间数据：
/* 1. FFT任务状态（高优先级，一次性执行） */
typedef enum {
    FFT_IDLE,        // 空闲（无抢占请求）
    FFT_REQUEST      // 有FFT抢占请求
} FFT_State;

/* 2. 波形刷新任务状态（低优先级，分步执行） */
typedef enum {
    WAVE_IDLE,       // 空闲
    WAVE_STEP1,      // 步骤1：读取ADC缓冲区数据
    WAVE_STEP2,      // 步骤2：计算波形显示坐标
    WAVE_STEP3,      // 步骤3：绘制单帧波形
    WAVE_STEP4       // 步骤4：刷新屏幕显示
} Wave_State;

/* 3. 全局状态与数据变量 */
FFT_State fft_state = FFT_IDLE;                // FFT任务状态
Wave_State wave_state = WAVE_IDLE;            // 波形任务状态
uint8_t wave_step_data[10];                   // 波形任务中间数据（抢占后恢复用）
#define ADC_BUFFER_SIZE 1024
uint32_t adc_buffer[2][ADC_BUFFER_SIZE] = {0};// DMA双缓冲缓冲区
DMA_HandleTypeDef hdma_adc1;                  // DMA句柄（需提前初始化）

2.2 高优先级中断：触发抢占请求
给 FFT 触发源（如按键）配置高优先级中断，中断中仅设置抢占请求，不执行复杂逻辑：
// 按键中断服务函数（优先级1，高）
void EXTI0_IRQHandler(void) {
    // 检查中断标志（避免误触发）
    if (__HAL_GPIO_EXTI_GET_FLAG(GPIO_PIN_0) != RESET) {
        __HAL_GPIO_EXTI_CLEAR_FLAG(GPIO_PIN_0); // 清除中断标志
        fft_state = FFT_REQUEST;                // 设置FFT抢占请求（1条指令）
    }
}

2.3 低优先级任务：分步执行 + 抢占检查
波形刷新任务按步骤执行，每步结束后检查 “FFT 抢占请求”，若有则暂停任务：
/**
 * 波形刷新任务（分步执行，非阻塞）
 * 功能：从ADC缓冲区读取数据→计算坐标→绘制波形→刷新屏幕
 */
void wave_task(void) {
    switch (wave_state) {
        case WAVE_IDLE:
            // 等待定时触发（如50ms刷新一次）
            if (HAL_GetTick() % 50 == 0) {
                wave_state = WAVE_STEP1; // 触发任务，进入第一步
            }
            break;
        
        case WAVE_STEP1:
            // 步骤1：读取ADC缓冲区（根据DMA的CT标志位选择可用缓冲区）
            if (hdma_adc1.Instance->CR & DMA_SxCR_CT) {
                wave_step_data[0] = (uint8_t)adc_buffer[0][0]; // 示例：读取缓冲区0首元素
            } else {
                wave_step_data[0] = (uint8_t)adc_buffer[1][0]; // 读取缓冲区1首元素
            }
            wave_state = WAVE_STEP2; // 进入下一步
            
            // 检查抢占：有FFT请求则暂停
            if (fft_state == FFT_REQUEST) {
                return;
            }
            break;
        
        case WAVE_STEP2:
            // 步骤2：计算波形显示坐标（示例：将ADC值映射为屏幕Y轴坐标）
            wave_step_data[1] = (wave_step_data[0] * 240) / 4095; // 12位ADC→240像素屏
            wave_state = WAVE_STEP3;
            
            if (fft_state == FFT_REQUEST) {
                return;
            }
            break;
        
        case WAVE_STEP3:
            // 步骤3：绘制单帧波形（拆分到单行，避免长耗时）
            LCD_DrawPixel(10, wave_step_data[1], RED); // 示例：绘制单个像素
            wave_state = WAVE_STEP4;
            
            if (fft_state == FFT_REQUEST) {
                return;
            }
            break;
        
        case WAVE_STEP4:
            // 步骤4：刷新屏幕（完成后回到空闲状态）
            LCD_Refresh();
            wave_state = WAVE_IDLE;
            break;
    }
}

2.4 高优先级任务：快速执行 + 中断屏蔽
FFT 任务执行时，屏蔽低优先级中断（避免被打扰），执行完成后恢复中断并清除请求：
/**
 * FFT分析任务（高优先级，抢占式）
 * 功能：读取最新ADC数据→执行FFT计算→输出结果
 */
void fft_task(void) {
    if (fft_state != FFT_REQUEST) {
        return; // 无请求则退出
    }
    
    // 1. 屏蔽低优先级中断（如波形刷新定时器中断，优先级2）
    HAL_NVIC_DisableIRQ(TIM6_DAC_IRQn);
    
    // 2. 读取最新ADC缓冲区（通过DMA的CT标志位判断可用缓冲区）
    uint32_t *p_available_buf = (hdma_adc1.Instance->CR & DMA_SxCR_CT)
                              ? adc_buffer[0] : adc_buffer[1];
    
    // 3. 执行FFT计算（示例：调用FFT库函数）
    float fft_result[ADC_BUFFER_SIZE/2];
    FFT_Compute(p_available_buf, fft_result, ADC_BUFFER_SIZE); // 自定义FFT函数
    
    // 4. 输出FFT结果（如显示到LCD或通过串口发送）
    LCD_DisplayFFT(fft_result);
    
    // 5. 恢复低优先级中断
    HAL_NVIC_EnableIRQ(TIM6_DAC_IRQn);
    
    // 6. 清除抢占请求，回到空闲状态
    fft_state = FFT_IDLE;
}

2.5 主循环：任务调度与低功耗
主循环按 “高优先级→低优先级” 顺序调度任务，空闲时进入低功耗模式（避免资源浪费）：
int main(void) {
    // 1. 初始化外设
    HAL_Init();
    SystemClock_Config();  // 系统时钟配置（如72MHz）
    MX_GPIO_Init();        // GPIO初始化（按键、LCD）
    MX_DMA_Init();         // DMA初始化（ADC双缓冲）
    MX_ADC1_Init();        // ADC初始化（连续转换模式）
    MX_TIM6_Init();        // 定时器初始化（波形刷新定时）
    LCD_Init();            // LCD屏幕初始化
    
    // 2. 配置中断优先级（FFT按键 > 波形定时器）
    HAL_NVIC_SetPriority(EXTI0_IRQn, 1, 0);    // FFT按键中断：优先级1
    HAL_NVIC_SetPriority(TIM6_DAC_IRQn, 2, 0); // 波形定时器中断：优先级2
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);            // 使能FFT按键中断
    HAL_NVIC_EnableIRQ(TIM6_DAC_IRQn);         // 使能波形定时器中断
    
    // 3. 启动ADC与DMA双缓冲
    HAL_DMAEx_MultiBufferStart_IT(&hdma_adc1,
                                  (uint32_t)&ADC1->DR,
                                  (uint32_t)adc_buffer[0],
                                  (uint32_t)adc_buffer[1],
                                  ADC_BUFFER_SIZE);
    __HAL_ADC_ENABLE_DMA(&hadc1);  // 使能ADC的DMA请求
    HAL_ADC_Start(&hadc1);         // 启动ADC连续转换
    
    // 4. 主循环：任务调度
    while (1) {
        // ① 先处理高优先级任务（FFT）
        fft_task();
        
        // ② 再处理低优先级任务（波形刷新）
        wave_task();
        
        // ③ 空闲时进入低功耗（WFI：等待中断唤醒）
        if (fft_state == FFT_IDLE && wave_state == WAVE_IDLE) {
            HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
        }
    }
}

三、架构优势
3.1 无资源浪费
低功耗：空闲时通过 HAL_PWR_EnterSLEEPMode 进入睡眠模式，仅中断可唤醒，降低功耗；
零冗余开销：无需 RTOS 的任务栈、调度器，仅用少量全局变量实现状态管理，内存占用 < 1KB。
3.2 支持抢占
快速响应：高优先级中断可打断低优先级任务，抢占延迟仅 “中断响应时间 + 步骤检查时间”（微秒级）；
灵活控制：通过中断优先级配置（NVIC）可自定义任务抢占顺序（如 “故障报警> FFT > 波形刷新”）。
3.3 高可靠性
断点恢复：任务中间数据和状态通过全局变量保存，抢占后可从断点继续执行，无进度丢失；
数据安全：通过 DMA 的 CT 标志位选择 “非采集中” 的缓冲区，避免数据读写冲突。
四、扩展与优化
4.1 新增任务
如需添加 “串口指令处理”（中优先级），仅需：
新增 UART_State 枚举记录串口任务状态；
编写 uart_task() 函数（分步执行：接收指令→解析→执行）；
在主循环中按 “FFT→串口→波形” 顺序调度。
4.2 优化抢占粒度
细粒度拆分：若波形刷新步骤耗时较长（如 WAVE_STEP3 绘制多像素），可进一步拆分为 “绘制 1 行→检查抢占→绘制下一行”，减少抢占延迟；
动态优先级：通过全局变量动态修改 fft_state 触发条件（如 “仅在按键长按 300ms 后触发 FFT”），避免误抢占。
五、复制后代码无颜色？解决办法
若复制到编辑器后仍无语法高亮，按以下步骤处理：
确认编辑器支持语法高亮：
推荐使用 Typora（Markdown 专用）、VS Code（安装 “C/C++” 插件）、Notion（需手动开启代码块语言）。
手动指定代码块语言：
若复制后代码块开头为 ，手动改为c （如将 替换为c ），编辑器会自动识别 C 语言并高亮。
VS Code 额外设置：
打开代码文件后，右下角点击 “Plain Text”，在弹出菜单中选择 “C”，即可触发语法颜色。
六、总结
该架构通过 “中断优先级控制 + 任务拆分 + 状态管理”，在裸机环境下实现了 “低资源占用” 与 “高实时抢占” 的平衡，特别适合 资源有限的嵌入式设备（如 STM32F1/F4、51 单片机），可广泛应用于示波器、工业控制器、智能传感器等场景。
核心逻辑：“中断触发请求，主循环分步执行，抢占时保存状态”，既避免了 RTOS 的复杂性，又解决了传统前后台架构 “无法抢占” 的痛点。
