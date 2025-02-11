# SOLAR-SEA-BOUY-SYSTEM

#include "main.h"
#include <stdio.h>
#include <string.h>

// Function Prototypes
void SystemClock_Config(void);
void UART_Transmit(char *msg);
void Read_DHT22(void);
void Read_pH_Sensor(void);
void Send_Alert(char *alert);

// Peripheral Handlers
UART_HandleTypeDef huart1;  // RFID & 433MHz Transmitter
UART_HandleTypeDef huart2;  // Barcode Scanner
ADC_HandleTypeDef hadc1;    // pH Sensor

// DHT22 Data Pin
#define DHT22_PORT GPIOB
#define DHT22_PIN GPIO_PIN_5

// Buzzer & LED
#define BUZZER_PORT GPIOB
#define BUZZER_PIN GPIO_PIN_12
#define LED_PORT GPIOB
#define LED_PIN GPIO_PIN_13

void HAL_UART_MspInit(UART_HandleTypeDef *huart) {
    // Configure UART GPIOs here
}

void HAL_ADC_MspInit(ADC_HandleTypeDef *hadc) {
    // Configure ADC GPIOs here
}

int main(void) {
    HAL_Init();
    SystemClock_Config();

    // Initialize UART
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 9600;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    HAL_UART_Init(&huart1);

    // Initialize ADC for pH Sensor
    hadc1.Instance = ADC1;
    hadc1.Init.ScanConvMode = DISABLE;
    hadc1.Init.ContinuousConvMode = ENABLE;
    HAL_ADC_Init(&hadc1);

    // Initialize GPIO for Buzzer & LED
    __HAL_RCC_GPIOB_CLK_ENABLE();
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = BUZZER_PIN | LED_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(BUZZER_PORT, &GPIO_InitStruct);

    while (1) {
        Read_DHT22();  // Read Temperature & Humidity
        Read_pH_Sensor();  // Read pH Level

        HAL_Delay(2000);
    }
}

// Function to Transmit Data over UART
void UART_Transmit(char *msg) {
    HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), 100);
}

// Function to Read DHT22 Sensor (Temperature & Humidity)
void Read_DHT22(void) {
    int temperature = 25;  // Mock Data, replace with actual reading logic
    int humidity = 60;  // Mock Data

    char buffer[50];
    sprintf(buffer, "Temp: %d C, Humidity: %d%%\r\n", temperature, humidity);
    UART_Transmit(buffer);

    if (temperature > 40) {
        Send_Alert("High Temp Alert! Heatwave Detected.");
    }
}

// Function to Read pH Sensor Value (Analog)
void Read_pH_Sensor(void) {
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
    float pH_value = (adc_value * 14.0) / 4095.0;  // Convert ADC value to pH Scale (0-14)

    char buffer[30];
    sprintf(buffer, "pH Level: %.2f\r\n", pH_value);
    UART_Transmit(buffer);

    if (pH_value < 5.0 || pH_value > 9.0) {
        Send_Alert("pH Alert! Unusual Water Acidity.");
    }
}

// Function to Send Alerts via 433MHz TX
void Send_Alert(char *alert) {
    UART_Transmit("ALERT: ");
    UART_Transmit(alert);
    UART_Transmit("\r\n");

    // Activate Warning Buzzer & LED
    HAL_GPIO_WritePin(BUZZER_PORT, BUZZER_PIN, GPIO_PIN_SET);
    HAL_GPIO_WritePin(LED_PORT, LED_PIN, GPIO_PIN_SET);
    HAL_Delay(3000);
    HAL_GPIO_WritePin(BUZZER_PORT, BUZZER_PIN, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED_PORT, LED_PIN, GPIO_PIN_RESET);
}

// Clock Configuration (Default)
void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK |
                                  RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2);
}
