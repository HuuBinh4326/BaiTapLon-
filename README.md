
#include "main.h"
#include "gpio.h"
#include "adc.h"
#include "i2c.h"
#include "i2c_lcd.h"
#include <string.h>

/* LCD I2C: dia chi 0x27 dich trai 1 bit = 0x4E */
I2C_LCD_HandleTypeDef lcd1 = {
    .hi2c = &hi2c1,
    .address = 0x4E
};

/* Nguong anh sang ADC 12-bit */
#define NGUONG_ANH_SANG 2500U

/* Chan dieu khien */
#define PIR_PORT       GPIOA
#define PIR_PIN        GPIO_PIN_1

#define LED_PORT       GPIOA
#define LED_PIN        GPIO_PIN_4

#define RELAY_QUAT_PORT GPIOA
#define RELAY_QUAT_PIN  GPIO_PIN_5

/* 1: relay kich HIGH; 0: relay kich LOW */
#define RELAY_ACTIVE_HIGH 1

void SystemClock_Config(void);
void Error_Handler(void);

static void LCD_GhiDong(uint8_t row, const char *text);
static uint32_t Doc_Anh_Sang(void);
static void DieuKhienQuat(GPIO_PinState pir);

/* Ghi toi da 16 ky tu len LCD */
static void LCD_GhiDong(uint8_t row, const char *text)
{
    char buffer[17];

    memset(buffer, ' ', 16);
    buffer[16] = '\0';
    strncpy(buffer, text, 16);

    lcd_gotoxy(&lcd1, 0, row);
    lcd_puts(&lcd1, buffer);
}

/* Doc gia tri ADC tai PA0 */
static uint32_t Doc_Anh_Sang(void)
{
    uint32_t value = 0;

    if (HAL_ADC_Start(&hadc1) == HAL_OK)
    {
        if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
        {
            value = HAL_ADC_GetValue(&hadc1);
        }

        HAL_ADC_Stop(&hadc1);
    }

    return value;
}

/* Dieu khien relay quat theo tin hieu PIR */
static void DieuKhienQuat(GPIO_PinState pir)
{
    GPIO_PinState relay_on;
    GPIO_PinState relay_off;

#if RELAY_ACTIVE_HIGH
    relay_on = GPIO_PIN_SET;
    relay_off = GPIO_PIN_RESET;
#else
    relay_on = GPIO_PIN_RESET;
    relay_off = GPIO_PIN_SET;
#endif

    if (pir == GPIO_PIN_SET)
    {
        HAL_GPIO_WritePin(
            RELAY_QUAT_PORT,
            RELAY_QUAT_PIN,
            relay_on
        );
    }
    else
    {
        HAL_GPIO_WritePin(
            RELAY_QUAT_PORT,
            RELAY_QUAT_PIN,
            relay_off
        );
    }
}

int main(void)
{
    uint32_t anh_sang;
    GPIO_PinState pir;
    GPIO_PinState den;

    GPIO_PinState pir_cu = GPIO_PIN_RESET;
    GPIO_PinState den_cu = GPIO_PIN_RESET;

    uint8_t lan_dau = 1;

    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_I2C1_Init();

    /* Khoi tao LED PA4 */
    HAL_GPIO_WritePin(LED_PORT, LED_PIN, GPIO_PIN_RESET);

    /* Dam bao quat tat luc khoi dong */
#if RELAY_ACTIVE_HIGH
    HAL_GPIO_WritePin(RELAY_QUAT_PORT, RELAY_QUAT_PIN, GPIO_PIN_RESET);
#else
    HAL_GPIO_WritePin(RELAY_QUAT_PORT, RELAY_QUAT_PIN, GPIO_PIN_SET);
#endif

    /* Khoi tao LCD */
    lcd_init(&lcd1);
    lcd_clear(&lcd1);

    //LCD_GhiDong(0, "PIR: KHOI DONG");
    //LCD_GhiDong(1, "DEN: DANG CHO");

    HAL_Delay(1000);

    while (1)
    {
        /* Doc cam bien anh sang PA0 */
        anh_sang = Doc_Anh_Sang();

        /* Doc cam bien PIR PA1 */
        pir = HAL_GPIO_ReadPin(PIR_PORT, PIR_PIN);

        /*
         * LOGIC DEN GIU NGUYEN:
         * ADC < 2500  -> LED PA4 TAT
         * ADC >= 2500 -> LED PA4 BAT
         */
        if (anh_sang < NGUONG_ANH_SANG)
        {
            den = GPIO_PIN_RESET;
        }
        else
        {
            den = GPIO_PIN_SET;
        }

        HAL_GPIO_WritePin(LED_PORT, LED_PIN, den);

        /* Dieu khien quat PA5 theo PIR PA1 */
        DieuKhienQuat(pir);

        /* LCD dong 1: trang thai PIR */
        if (lan_dau || pir != pir_cu)
        {
            if (pir == GPIO_PIN_SET)
            {
                LCD_GhiDong(0, "QUAT: BAT");
            }
            else
            {
                LCD_GhiDong(0, "QUAT: TAT");
            }

            pir_cu = pir;
        }

        /* LCD dong 2: trang thai den */
        if (lan_dau || den != den_cu)
        {
            if (den == GPIO_PIN_SET)
            {
                LCD_GhiDong(1, "DEN: BAT");
            }
            else
            {
                LCD_GhiDong(1, "DEN: TAT");
            }

            den_cu = den;
        }

        lan_dau = 0;

        HAL_Delay(100);
    }
}

/* Clock 72 MHz cho STM32F103C8T6 */
void SystemClock_Config(void)
{
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;

    /* HSE 8 MHz x 9 = 72 MHz */
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;

    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
        Error_Handler();
    }

    RCC_ClkInitStruct.ClockType =
        RCC_CLOCKTYPE_HCLK |
        RCC_CLOCKTYPE_SYSCLK |
        RCC_CLOCKTYPE_PCLK1 |
        RCC_CLOCKTYPE_PCLK2;

    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

    if (HAL_RCC_ClockConfig(
            &RCC_ClkInitStruct,
            FLASH_LATENCY_2) != HAL_OK)
    {
        Error_Handler();
    }
}

/* Xu ly loi he thong */
void Error_Handler(void)
{
    __disable_irq();

    while (1)
    {
    }
}

