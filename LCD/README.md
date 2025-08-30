# LCD test

STM32 HAL 라이브러리를 이용해서 ILI9341 LCD 를 제어하는 예제

다음의 사이트들을 참고하였다

- https://www.micropeta.com/video37#google_vignette
- https://www.youtube.com/watch?v=8hypVJ2U7OE
- https://github.com/eziya/STM32_HAL_ILI9341?tab=readme-ov-file
- https://blog.naver.com/eziya76/221579262995


## 환경 설정
NUCLEO-F446RE를 사용하였고 사용한 pin map 은 다음과 같다

| LCD 모듈 핀 | STM32F103 연결 예시     |
|-------------|-------------------------|
| VCC         | 3.3V                   |
| GND         | GND                    |
| CS          | PB2 (GPIO OUTPUT)      |
| RESET       | PB1 (GPIO OUTPUT)      |
| DC/RS       | PB14 (GPIO OUTPUT)     |
| SDI/MOSI    | PB15 (SPI2_MOSI)       |
| SCK         | PB13 (SPI2_SCK)        |
| LED         | 3.3V (백라이트)        |
| SD0/MISO    | -                      |

<img width="1385" height="1126" alt="image" src="https://github.com/user-attachments/assets/62da687c-66b0-42d7-a7f7-2d77c00df3de" />

## CODE
### 추가/변경한 기능 정리
기본적으로 https://www.micropeta.com/video37#google_vignette 에서 제공하는 파일을 베이스로 했고, 필요한 기능을 더하고 환경에 맞게 설정을 수정했다

- ILI9341 드라이버 파일 안에 화면 4분할 함수, 센서 값과 단위 표시하는 함수, 구조체로 받아온 값 너비 계산에서 출력하는 함수 생성
- 원래 예제 링크에서 stm32F1xx 를 위한 파일이었으나 f446에서 돌아가도록 수정
- 핀맵 예쁘게 변경
  - F446에서 LCD에 사용되는 모든 핀들이 널부러져있지 않고 예쁘게 일자로 꽂아져있다!

## RTOS
MODBUS를 통한 센서 데이터들을 LCD에 표시하기 위해 구조체를 통해 센서값들을 받아오고 센서 네개의 값들을 화면 네개에 비율 맞춰서 예쁘게 분할해서 표시
ILI9341 드라이버 위에 4분할 화면과 데이터를 표시하기 위한 함수를 여러가지 제작했다.

아래는 RTOS환경에서 돌아가는 LCD task의 코드이다
```C
void StartLCD(void *argument)
{
  /* USER CODE BEGIN StartLCD */
	ILI9341_Init();
	ILI9341_FillScreen(WHITE);
	ILI9341_SetRotation(SCREEN_HORIZONTAL_2);
	ILI9341_DrawText("MADE BY JIHWAN & HYESUNG", FONT3, 40, 110, BLACK, WHITE);
	osDelay(500);

	ILI9341_FillScreen(WHITE);
	Draw_Padded_Quadrant_Lines(20, NAVY);
	Draw_Quadrant_Titles(FONT4, 10, BLUE, WHITE); //10만큼 위쪽 여백주기
	char* sensor_items[] = {"CO2:", "Temp:", "Humid:", "PM2.5:"};
	char* sensor_units[] = {"ppm", "C", "%RH", "ug/m3"};
	Draw_Static_Sensor_Layout(sensor_items, sensor_units, 4, FONT3, 10, BLACK, WHITE);

    // --- 2. 데이터 수신 및 문자열 변환에 사용할 변수들 ---
    SensorData_t receivedData; // 큐에서 받을 구조체 변수
    osStatus_t status;         // 큐 수신 상태 확인용 변수

    // 화면에 표시할 문자열을 담을 버퍼
    char co2_str[10];
    char temp_str[10];
    char humid_str[10];
    char pm25_str[10];

  /* Infinite loop */
  for(;;)
  {
	// --- 3. 큐에서 데이터가 올 때까지 무한정 대기 ---
	//    - osWaitForever: 데이터가 수신될 때까지 태스크는 Block 상태가 되어 CPU를 소모하지 않음
	status = osMessageQueueGet(ModbusDataHandle, &receivedData, NULL, osWaitForever);

	// --- 4. 데이터 수신 성공 시 화면 업데이트 ---
	if (status == osOK)
	{
		// 수신된 숫자 데이터를 문자열로 변환 (sprintf 사용)
		sprintf(co2_str, "%u", receivedData.co2);
		sprintf(temp_str, "%.1f", receivedData.temperature); // 소수점 한 자리
		sprintf(humid_str, "%.1f", receivedData.humidity);   // 소수점 한 자리
		sprintf(pm25_str, "%u", receivedData.pm25);

		// Update_Sensor_Values 함수에 맞게 char* 배열 생성
		char* sensor_values[] = {co2_str, temp_str, humid_str, pm25_str};
		Update_Sensor_Values(receivedData.slaveId, sensor_values, sensor_units, 4, FONT3, 10, RED, WHITE);
	}


    //osDelay(1);
  }
  /* USER CODE END StartLCD */
}
```

이 폴더에 포함된 드라이버 파일과 main코드 안에 task를 추가하고 나면 LCD에는 다음과 같은 화면이 표시된다

![IMG_7908](https://github.com/user-attachments/assets/ed0045ef-c6f5-49e0-b444-c05da64ce074)

