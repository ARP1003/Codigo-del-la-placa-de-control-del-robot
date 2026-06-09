#include <WiFi.h>
#include <esp_now.h>

// PWM PARA EL BTS7960

#define RPWM_IZQ      19
#define LPWM_IZQ      18

#define RPWM_DER      25
#define LPWM_DER      26

// PWM LIMPIEZA

#define PWM_BOMBA     17
#define PWM_RODILLO   16

// ENCODERS

#define ENC_IZQ_A     22
#define ENC_IZQ_B     23

#define ENC_DER_A     32
#define ENC_DER_B     33

// FINALES DE CARRERA

#define FC1           27
#define FC2           13

// PWM 

#define PWM_FREQ      10000
#define PWM_RES       8

// WATCHDOG

unsigned long ultimoPaquete = 0;

#define TIMEOUT_CONEXION 500

bool conexionActiva = false;

// RAMPA PWM

int pwmActual = 0;

int pwmObjetivo = 0;

const int PASO_RAMPA = 5;

// SENSOR DELANTERO SEGURIDAD

bool sensorDelanteroActivo = false;

bool bloqueoAvance = false;

bool bloqueoReversa = false;

unsigned long tiempoSinPanel = 0;

#define TIEMPO_MAX_SIN_PANEL 4000

#define PWM_SEGURIDAD 125

// SECUENCIA LIMPIEZA

bool secuenciaLimpiezaActiva = false;

bool limpiezaEncendida = false;

unsigned long tiempoSecuencia = 0;

#define ESPERAR_RODILLO 1000
#define ESPERAR_BOMBA   500

enum EstadoLimpieza
{
    LIMPIEZA_OFF,
    ENCENDIENDO_BOMBA,
    ENCENDIENDO_RODILLO,
    LIMPIEZA_ON,
    APAGANDO_RODILLO,
    APAGANDO_BOMBA
};

EstadoLimpieza estadoLimpieza =
LIMPIEZA_OFF;

// DIRECCIONES

#define STOP        0
#define ADELANTE    1
#define ATRAS       2
#define IZQUIERDA   3
#define DERECHA     4

// MODOS

#define MANUAL      0
#define MTTO        1

// ESTRUCTURA DATOS

typedef struct
{
    int velocidad;

    int direccion;

    bool motorON;

    bool limpiezaON;

    bool bombaON;

    bool rodilloON;

    int modo;

} DatosControl;

// VARIABLE GLOBAL

DatosControl datos;

// CALLBACK RX

void OnDataRecv(const esp_now_recv_info_t *info,
                const uint8_t *incomingData,
                int len)
{
    memcpy(&datos,
           incomingData,
           sizeof(datos));

    ultimoPaquete = millis();

    conexionActiva = true;

    Serial.println();
    Serial.println("========== RX ==========");

    // MODO

    Serial.print("Modo: ");

    if (datos.modo == MANUAL)
    {
        Serial.println("MANUAL");
    }
    else
    {
        Serial.println("MTTO");
    }

    // DIRECCION

    Serial.print("Direccion: ");

    switch (datos.direccion)
    {
        case STOP:
            Serial.println("STOP");
        break;

        case ADELANTE:
            Serial.println("ADELANTE");
        break;

        case ATRAS:
            Serial.println("ATRAS");
        break;

        case IZQUIERDA:
            Serial.println("IZQUIERDA");
        break;

        case DERECHA:
            Serial.println("DERECHA");
        break;
    }

    // VARIABLES

    Serial.print("Velocidad: ");
    Serial.println(datos.velocidad);

    Serial.print("Motor: ");
    Serial.println(datos.motorON);

    Serial.print("Limpieza: ");
    Serial.println(datos.limpiezaON);

    Serial.print("Bomba: ");
    Serial.println(datos.bombaON);

    Serial.print("Rodillo: ");
    Serial.println(datos.rodilloON);

    Serial.println("========================");
}

// SETUP

void setup()
{
    Serial.begin(115200);

    // PINES

    pinMode(FC1, INPUT_PULLUP);

    pinMode(FC2, INPUT_PULLUP);

    pinMode(ENC_IZQ_A, INPUT);

    pinMode(ENC_IZQ_B, INPUT);

    pinMode(ENC_DER_A, INPUT);

    pinMode(ENC_DER_B, INPUT);

    // PWM

    ledcAttach(RPWM_IZQ,
               PWM_FREQ,
               PWM_RES);

    ledcAttach(LPWM_IZQ,
               PWM_FREQ,
               PWM_RES);

    ledcAttach(RPWM_DER,
               PWM_FREQ,
               PWM_RES);

    ledcAttach(LPWM_DER,
               PWM_FREQ,
               PWM_RES);

    ledcAttach(PWM_BOMBA,
               PWM_FREQ,
               PWM_RES);

    ledcAttach(PWM_RODILLO,
               PWM_FREQ,
               PWM_RES);

    // STOP INICIAL

    stopTotal();

    datos.motorON = false;

    datos.limpiezaON = false;

    datos.bombaON = false;

    datos.rodilloON = false;

    datos.direccion = STOP;

    datos.velocidad = 0;

    // WIFI

    WiFi.mode(WIFI_STA);

    WiFi.disconnect();

    Serial.print("MAC ROBOT: ");

    Serial.println(WiFi.macAddress());

    // ESP NOW

    if (esp_now_init() != ESP_OK)
    {
        Serial.println("ERROR ESP NOW");

        return;
    }

    esp_now_register_recv_cb(OnDataRecv);

    Serial.println("ESP NOW RX LISTO");
}

// LOOP

void loop()
{
    watchdogConexion();

    // VALIDAR CONEXION

    if (conexionActiva)
    {
        // MODO MANUAL

        if (datos.modo == MANUAL)
        {
            // TRACCION

            if (datos.motorON)
            {
                // LOGICA SEGUN DIRECCION

                switch (datos.direccion)
                {
                    // ADELANTE -> FC1

                    case ADELANTE:

                        sensorDelanteroActivo =
                        !digitalRead(FC1);

                        if (sensorDelanteroActivo)
                        {
                            bloqueoAvance = false;

                            tiempoSinPanel = 0;
                        }
                        else
                        {
                            if (tiempoSinPanel == 0)
                            {
                                tiempoSinPanel = millis();
                            }

                            if (millis() - tiempoSinPanel >=
                                TIEMPO_MAX_SIN_PANEL)
                            {
                                bloqueoAvance = true;
                            }
                        }

                    break;

                    // ATRAS -> FC2

                    case ATRAS:

                        sensorDelanteroActivo =
                        !digitalRead(FC2);

                        if (sensorDelanteroActivo)
                        {
                            bloqueoReversa = false;

                            tiempoSinPanel = 0;
                        }
                        else
                        {
                            if (tiempoSinPanel == 0)
                            {
                                tiempoSinPanel = millis();
                            }

                            if (millis() - tiempoSinPanel >=
                                TIEMPO_MAX_SIN_PANEL)
                            {
                                bloqueoReversa = true;
                            }
                        }

                    break;

                    // GIROS -> IGNORAR SENSORES

                    case IZQUIERDA:

                    case DERECHA:

                    case STOP:

                        tiempoSinPanel = 0;

                    break;
                }

                // VELOCIDAD NORMAL

                pwmObjetivo =
                map(datos.velocidad,
                    0,
                    100,
                    0,
                    255);

                // VELOCIDAD SEGURIDAD ADELANTE

                if (datos.direccion == ADELANTE)
                {
                    if (digitalRead(FC1))
                    {
                        pwmObjetivo =
                        PWM_SEGURIDAD;
                    }
                }

                // VELOCIDAD SEGURIDAD ATRAS

                else if (datos.direccion == ATRAS)
                {
                    if (digitalRead(FC2))
                    {
                        pwmObjetivo =
                        PWM_SEGURIDAD;
                    }
                }

                actualizarRampaPWM();

                // CONTROL DIRECCION

                switch (datos.direccion)
                {
                    // ADELANTE

                    case ADELANTE:

                        if (bloqueoAvance)
                        {
                            stopMotores();
                        }
                        else
                        {
                            moverAdelante(pwmActual);
                        }

                    break;

                    // ATRAS

                    case ATRAS:

                        if (bloqueoReversa)
                        {
                            stopMotores();
                        }
                        else
                        {
                            moverAtras(pwmActual);
                        }

                    break;

                    // IZQUIERDA

                    case IZQUIERDA:

                        girarIzquierda(pwmActual);

                    break;

                    // DERECHA

                    case DERECHA:

                        girarDerecha(pwmActual);

                    break;

                    // STOP

                    default:

                        stopMotores();

                    break;
                }
            }
            else
            {
                stopMotores();

                pwmActual = 0;

                pwmObjetivo = 0;
            }

            // LIMPIEZA AUTOMATICA

            if (datos.limpiezaON)
            {
                if (!limpiezaEncendida)
                {
                    limpiezaEncendida = true;

                    estadoLimpieza =
                    ENCENDIENDO_BOMBA;

                    tiempoSecuencia = millis();
                }
            }
            else
            {
                if (limpiezaEncendida)
                {
                    limpiezaEncendida = false;

                    estadoLimpieza =
                    APAGANDO_RODILLO;

                    tiempoSecuencia = millis();
                }
            }

            actualizarSecuenciaLimpieza();
        }

        // MODO MTTO

        else if (datos.modo == MTTO)
        {
            // TRACCION MTTO

            switch (datos.direccion)
            {
                case ADELANTE:
                    moverAdelante(255);
                break;

                case ATRAS:
                    moverAtras(255);
                break;

                default:
                    stopMotores();
                break;
            }

            // BOMBA

            if (datos.bombaON)
            {
                activarBomba();
            }
            else
            {
                apagarBomba();
            }

            // RODILLO

            if (datos.rodilloON)
            {
                activarRodillo();
            }
            else
            {
                apagarRodillo();
            }
        }
    }
    else
    {
        stopMotores();

        apagarBomba();

        apagarRodillo();
    }
}

// PWM INVERTIDO

int pwmInvertido(int valor)
{
    valor = constrain(valor, 0, 255);

    return 255 - valor;
}

// ACTUALIZAR RAMPA PWM

void actualizarRampaPWM()
{
    if (pwmActual < pwmObjetivo)
    {
        pwmActual += PASO_RAMPA;

        if (pwmActual > pwmObjetivo)
        {
            pwmActual = pwmObjetivo;
        }
    }

    else if (pwmActual > pwmObjetivo)
    {
        pwmActual -= PASO_RAMPA;

        if (pwmActual < pwmObjetivo)
        {
            pwmActual = pwmObjetivo;
        }
    }
}

// SECUENCIA LIMPIEZA

void actualizarSecuenciaLimpieza()
{
    switch (estadoLimpieza)
    {
        // OFF

        case LIMPIEZA_OFF:

            apagarBomba();

            apagarRodillo();

        break;

        // ENCENDIENDO BOMBA

        case ENCENDIENDO_BOMBA:

            activarBomba();

            if (millis() - tiempoSecuencia >=
                ESPERAR_RODILLO)
            {
                estadoLimpieza =
                ENCENDIENDO_RODILLO;

                tiempoSecuencia = millis();
            }

        break;

        // ENCENDIENDO RODILLO

        case ENCENDIENDO_RODILLO:

            activarBomba();

            activarRodillo();

            estadoLimpieza = LIMPIEZA_ON;

        break;

        // LIMPIEZA ON

        case LIMPIEZA_ON:

            activarBomba();

            activarRodillo();

        break;

        // APAGANDO RODILLO

        case APAGANDO_RODILLO:

            apagarRodillo();

            activarBomba();

            if (millis() - tiempoSecuencia >=
                ESPERAR_BOMBA)
            {
                estadoLimpieza =
                APAGANDO_BOMBA;
            }

        break;

        // APAGANDO BOMBA

        case APAGANDO_BOMBA:

            apagarRodillo();

            apagarBomba();

            estadoLimpieza = LIMPIEZA_OFF;

        break;
    }
}

// STOP MOTORES

void stopMotores()
{
    ledcWrite(RPWM_IZQ, 255);
    ledcWrite(LPWM_IZQ, 255);

    ledcWrite(RPWM_DER, 255);
    ledcWrite(LPWM_DER, 255);
}

// ADELANTE

void moverAdelante(int pwm)
{
    int pwmReal = pwmInvertido(pwm);

    ledcWrite(RPWM_IZQ, pwmReal);
    ledcWrite(LPWM_IZQ, 255);

    ledcWrite(RPWM_DER, pwmReal);
    ledcWrite(LPWM_DER, 255);
}

// ATRAS

void moverAtras(int pwm)
{
    int pwmReal = pwmInvertido(pwm);

    ledcWrite(RPWM_IZQ, 255);
    ledcWrite(LPWM_IZQ, pwmReal);

    ledcWrite(RPWM_DER, 255);
    ledcWrite(LPWM_DER, pwmReal);
}

// IZQUIERDA

void girarIzquierda(int pwm)
{
    int pwmReal = pwmInvertido(pwm);

    ledcWrite(RPWM_IZQ, 255);
    ledcWrite(LPWM_IZQ, pwmReal);

    ledcWrite(RPWM_DER, pwmReal);
    ledcWrite(LPWM_DER, 255);
}

// DERECHA


void girarDerecha(int pwm)
{
    int pwmReal = pwmInvertido(pwm);

    ledcWrite(RPWM_IZQ, pwmReal);
    ledcWrite(LPWM_IZQ, 255);

    ledcWrite(RPWM_DER, 255);
    ledcWrite(LPWM_DER, pwmReal);
}

// BOMBA ON

void activarBomba()
{

    ledcWrite(PWM_BOMBA, 255);
}

// BOMBA OFF

void apagarBomba()
{
    ledcWrite(PWM_BOMBA, 0);
}

// RODILLO ON

void activarRodillo()
{
    ledcWrite(PWM_RODILLO, 255);
}

// RODILLO OFF

void apagarRodillo()
{
    ledcWrite(PWM_RODILLO, 0);
}

// WATCHDOG

void watchdogConexion()
{
    if (conexionActiva)
    {
        if (millis() - ultimoPaquete >
            TIMEOUT_CONEXION)
        {
            conexionActiva = false;

            Serial.println();
            Serial.println("TIMEOUT");

            // STOP TOTAL

            stopTotal();

            // LIMPIAR COMANDOS

            datos.motorON = false;

            datos.limpiezaON = false;

            datos.bombaON = false;

            datos.rodilloON = false;

            datos.direccion = STOP;

            datos.velocidad = 0;
        }
    }
}

// STOP TOTAL

void stopTotal()
{
    stopMotores();

    ledcWrite(PWM_BOMBA, 0);

    ledcWrite(PWM_RODILLO, 0);

    Serial.println("STOP TOTAL");
}
