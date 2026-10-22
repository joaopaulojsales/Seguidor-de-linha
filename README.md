# Seguidor-de-linha
// ==========================================
// CÓDIGO FINAL - FLUIDEZ NA RETA + RIGIDEZ NA CURVA
// ==========================================

// --- Definição dos Pinos dos Sensores ---
const int sensorDir = 2; // Sensor da Direita
const int sensorEsq = 3; // Sensor da Esquerda

// --- Definição dos pinos da Ponte H (L298N) ---
const int enA = 5;  // Motor Esquerdo (COM PWM)
const int in1 = 10;
const int in2 = 11;
const int in3 = 13;
const int in4 = 12;
const int enB = 9;  // Motor Direito (COM PWM)

// --- Definição de cores ---
const int PRETO = HIGH;  
const int BRANCO = LOW;  

// === VARIÁVEIS DE VELOCIDADE E FORÇA ===
int velBase = 140;           // FLUIDEZ AUMENTADA: Mais solto e rápido na reta
int velCorrecao = 150;       // RIGIDEZ: Força bruta da marcha ré para corrigir a curva
int velBusca = 140;          // RADAR: Velocidade do giro de busca em leque
int velMinimaParaPulso = 60; // Limiar do pulso

// === VARIÁVEIS DA MÁQUINA DE ESTADOS E TEMPO ===
enum Estado {
  NA_PISTA,
  RECUANDO,
  BUSCANDO
};
Estado estadoAtual = NA_PISTA;

unsigned long ultimoTempoLinha = 0; 
const unsigned long TEMPO_PERDIDO = 1000; // 1 segundo no branco = ativa a busca

unsigned long tempoInicioEstado = 0;
unsigned long tempoAtualSweep = 200; 
int dirSweep = 1; // Memória do último lado que viu a fita (1 = Esq, -1 = Dir)

void setup() {
  Serial.begin(9600);
  delay(1000);

  pinMode(sensorDir, INPUT);
  pinMode(sensorEsq, INPUT);

  pinMode(enA, OUTPUT);
  pinMode(enB, OUTPUT);
  pinMode(in1, OUTPUT);
  pinMode(in2, OUTPUT);
  pinMode(in3, OUTPUT);
  pinMode(in4, OUTPUT);
}

void loop() {
  int leituraDir = digitalRead(sensorDir);
  int leituraEsq = digitalRead(sensorEsq);
  unsigned long tempoAtual = millis();

  // Se encostou na fita, ZERA O CRONÔMETRO e garante o estado NA_PISTA
  if (leituraDir == PRETO || leituraEsq == PRETO) {
    ultimoTempoLinha = tempoAtual;
    estadoAtual = NA_PISTA; 
  }

  // ==========================================
  // ESTADO: NA PISTA (Movimento Normal)
  // ==========================================
  if (estadoAtual == NA_PISTA) {
    
    // Verifica se perdeu a pista (passou de 1 segundo totalmente no branco)
    if (tempoAtual - ultimoTempoLinha > TEMPO_PERDIDO) {
      estadoAtual = RECUANDO; 
      tempoInicioEstado = tempoAtual;
    } 
    else {
      // 1. TUDO BRANCO -> Fita perfeita no meio (MOVIMENTO FLUIDO)
      if (leituraDir == BRANCO && leituraEsq == BRANCO) {
        moverMotores(velBase, velBase); 
      } 
      // 2. CORREÇÃO RÍGIDA DIREITA
      else if (leituraDir == PRETO && leituraEsq == BRANCO) {
        moverMotores(velBase, -velCorrecao); 
        dirSweep = -1; // Memória para a busca (Direita)
      } 
      // 3. CORREÇÃO RÍGIDA ESQUERDA
      else if (leituraDir == BRANCO && leituraEsq == PRETO) {
        moverMotores(-velCorrecao, velBase); 
        dirSweep = 1;  // Memória para a busca (Esquerda)
      } 
      // 4. CRUZAMENTO
      else if (leituraDir == PRETO && leituraEsq == PRETO) {
        moverMotores(velBase, velBase);
      }
    }
  }

  // ==========================================
  // ESTADO DE EMERGÊNCIA: RADAR DE BUSCA INTELIGENTE
  // ==========================================
  else if (estadoAtual == RECUANDO) {
    moverMotores(-velBusca, -velBusca); // Breve marcha ré
    
    if (tempoAtual - tempoInicioEstado > 150) { 
      estadoAtual = BUSCANDO;
      tempoInicioEstado = tempoAtual;
      tempoAtualSweep = 200; 
    }
  } 
  
  else if (estadoAtual == BUSCANDO) {
    if (dirSweep == 1) {
      moverMotores(-velBusca, velBusca); // Radar Esquerda
    } else {
      moverMotores(velBusca, -velBusca); // Radar Direita
    }

    if (tempoAtual - tempoInicioEstado > tempoAtualSweep) {
      dirSweep = -dirSweep;           
      tempoAtualSweep += 150;         
      tempoInicioEstado = tempoAtual; 
    }
  }
}

// --- Função com Kickstart Otimizado (Mais suave na reta) ---
void moverMotores(int vEsqRequisitada, int vDirRequisitada) {
  
  int pwmEsq = vEsqRequisitada;
  int pwmDir = vDirRequisitada;

  // Direção Roda Esquerda
  if (pwmEsq >= 0) {
    digitalWrite(in1, HIGH); digitalWrite(in2, LOW);
  } else {
    digitalWrite(in1, LOW); digitalWrite(in2, HIGH);
    pwmEsq = -pwmEsq; 
  }

  // Direção Roda Direita
  if (pwmDir >= 0) {
    digitalWrite(in3, HIGH); digitalWrite(in4, LOW);
  } else {
    digitalWrite(in3, LOW); digitalWrite(in4, HIGH);
    pwmDir = -pwmDir; 
  }

  if (pwmEsq > 255) pwmEsq = 255;
  if (pwmDir > 255) pwmDir = 255;

  static int ultimaVelReqEsq = 0;
  static int ultimaVelReqDir = 0;

  bool inverteuEsq = (ultimaVelReqEsq > 0 && vEsqRequisitada < 0) || (ultimaVelReqEsq < 0 && vEsqRequisitada > 0);
  bool inverteuDir = (ultimaVelReqDir > 0 && vDirRequisitada < 0) || (ultimaVelReqDir < 0 && vDirRequisitada > 0);
  
  bool precisaArranqueEsq = inverteuEsq || (abs(ultimaVelReqEsq) < velMinimaParaPulso && pwmEsq >= velMinimaParaPulso);
  bool precisaArranqueDir = inverteuDir || (abs(ultimaVelReqDir) < velMinimaParaPulso && pwmDir >= velMinimaParaPulso);

  if (precisaArranqueEsq || precisaArranqueDir) {
    if (precisaArranqueEsq) analogWrite(enA, 255);
    else analogWrite(enA, pwmEsq);

    if (precisaArranqueDir) analogWrite(enB, 255);
    else analogWrite(enB, pwmDir);

    // TEMPO REDUZIDO PARA 10ms: Elimina o "soluço" quando volta para a linha reta!
    delay(10); 
  }

  analogWrite(enA, pwmEsq);
  analogWrite(enB, pwmDir);

  ultimaVelReqEsq = vEsqRequisitada;
  ultimaVelReqDir = vDirRequisitada;
}
