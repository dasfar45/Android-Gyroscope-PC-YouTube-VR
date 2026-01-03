# Android Gyroscope PC YouTube VR
Este projeto permite transformar um smartphone Android em um controle giroscópico (VR) no PC para vídeos 360° no YouTube

1. Pré-requisitos

    Celular Android: Instale o app Sensor Server (Créditos aos desenvolvedores originais).

    Computador: Tenha o Python instalado e o navegador com a extensão Tampermonkey.


2. Instalação (Python)

No terminal do seu PC, instale as bibliotecas necessárias:

```batch
pip install websocket-client pyautogui keyboard
```


3. Baixar Sensor Server.apk

    Abra o Sensor Server.

    Vá em Server e ative o botão de "Start".

    Anote o IP e a Porta que aparecem no app (ex:192.168.0.103:8080)

   4: Instalar Extensão (Tampermonkey)
```
www.tampermonkey.net
```
  Adicione um novo script no Tampermonkey.


```
// ==UserScript==
// @name         YouTube VR - Remover Botões Da Tela Do Video
// @namespace    http://tampermonkey.net/
// @version      16.2
// @match        https://www.youtube.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';
    let vrMode = false;
    let cursorVisivel = false;

    const style = document.createElement('style');
    style.id = 'vr-fit-style';
    style.innerHTML = `
        #movie_player, .html5-video-container, video {
            width: 100vw !important;
            height: 100vh !important;
            position: fixed !important;
            top: 50% !important;
            left: 50% !important;
            transform: translate(-50%, -50%) scale(1.0) !important;
            z-index: 999999 !important;
            object-fit: fill !important;
            background: black !important;
        }
        .ytp-chrome-bottom, .ytp-gradient-bottom, .ytp-chrome-top, .ytp-gradient-top, .ytp-pause-overlay {
            display: none !important;
        }
        body { overflow: hidden !important; background: black !important; }
    `;

    const cursorHide = document.createElement('style');
    cursorHide.innerHTML = `* { cursor: none !important; }`;

    const preventDblClick = (e) => {
        if (vrMode) {
            e.stopPropagation();
            e.preventDefault();
        }
    };

    document.addEventListener('keydown', (e) => {
        const key = e.key.toLowerCase();

        // MUDANÇA: Agora usa a tecla "Insert" para evitar conflito com atalhos do YouTube
        if (e.key === 'Insert') {
            vrMode = !vrMode;
            if (vrMode) {
                document.head.appendChild(style);
                document.head.appendChild(cursorHide);
                document.addEventListener('dblclick', preventDblClick, true);
                window.dispatchEvent(new Event('resize'));
            } else {
                location.reload();
            }
        }

        if (key === 'q' && vrMode) {
            cursorVisivel = !cursorVisivel;
            if (cursorVisivel) {
                if (document.head.contains(cursorHide)) document.head.removeChild(cursorHide);
            } else {
                document.head.appendChild(cursorHide);
            }
        }
    });
})();
```
Salve e deixe ativado.


5. Instalar Python
   
   No código Python, atualize a variável IP_CELULAR com o endereço IP do seu telefone que aparece no Sensor Server.
   
   Salve e Rode esse Codigo com Python
```
import websocket
import json
import pyautogui
import keyboard
import threading
import time

IP_CELULAR = "192.168.0.103"
PORTA = "8080"

pyautogui.PAUSE = 0
pyautogui.FAILSAFE = False

class ControleVRFisico:
    def __init__(self):
        self.sensi_x = 18  
        self.sensi_y = 12
        self.ativo = False
        
        self.target_x = 0
        self.target_y = 0
        self.current_x = 0
        self.current_y = 0
        self.suavidade = 0.25
        
        self.screen_w, self.screen_h = pyautogui.size()
        self.centro_x = self.screen_w // 2
        self.centro_h = self.screen_h // 2

    def toggle_controle(self):
        if not self.ativo:
            self.ativo = True
            print("\n[ATIVADO] Modo Horizontal Corrigido...")
            pyautogui.moveTo(self.centro_x, self.centro_h)
            pyautogui.mouseDown()
        else:
            self.ativo = False
            pyautogui.mouseUp()
            self.target_x = self.target_y = self.current_x = self.current_y = 0
            print("\n[DESATIVADO] Mouse liberado.")

    def on_message(self, ws, message):
        if not self.ativo: return
        try:
            data = json.loads(message)
            v = data.get('values', [0, 0, 0])
            
            if abs(v[0]) < 0.04 and abs(v[1]) < 0.04:
                self.target_x = self.target_y = 0
            else:
                # CORREÇÃO DOS EIXOS:
                # Se Esquerda/Direita fazia subir/descer, o sinal vinha de v[1] para o Y.
                # Se Cima/Baixo fazia ir para os lados, o sinal vinha de v[0] para o X.
                # Inverti a atribuição e os sinais conforme seu relato:
                self.target_x = v[0] * self.sensi_x  
                self.target_y = -v[1] * self.sensi_y 
        except:
            pass

    def loop_de_movimento(self):
        while True:
            if self.ativo:
                self.current_x += (self.target_x - self.current_x) * self.suavidade
                self.current_y += (self.target_y - self.current_y) * self.suavidade

                if abs(self.current_x) > 0.1 or abs(self.current_y) > 0.1:
                    pyautogui.moveRel(self.current_x, self.current_y, _pause=False)
                
                mx, my = pyautogui.position()
                if mx < 300 or mx > self.screen_w - 300 or my < 200 or my > self.screen_h - 200:
                    pyautogui.mouseUp()
                    pyautogui.moveTo(self.centro_x, self.centro_h)
                    time.sleep(0.01)
                    pyautogui.mouseDown()

            time.sleep(0.016)

    def iniciar_ws(self):
        url = f"ws://{IP_CELULAR}:{PORTA}/sensor/connect?type=android.sensor.gyroscope"
        ws = websocket.WebSocketApp(url, on_message=self.on_message)
        ws.run_forever()

if __name__ == "__main__":
    controle = ControleVRFisico()
    
    t_ws = threading.Thread(target=controle.iniciar_ws, daemon=True)
    t_ws.start()
    
    t_move = threading.Thread(target=controle.loop_de_movimento, daemon=True)
    t_move.start()
    
    print("INSERT: Ativar/Desativar VR | ESC: Sair")
    keyboard.add_hotkey('insert', controle.toggle_controle)
    
    try:
        keyboard.wait('esc')
    finally:
        pyautogui.mouseUp()
        print("Programa encerrado.")
```

  

   Com o Sensor Server ativado, Com o Codigo Python e a Extensão ja rodando, Só entrar no Video 360 graus desejado, Abrir em Tela Cheia, Clicar no Centro do Video e Apertar a Tecla "Insert"
     
   Segure o celular na Horizontal (como um controle).
    
   Para Parar o Modo VR, Só apertar a tecla "Insert" e Apertar a Letra "Q" para Liberar o Mouse

     
⌨️ Atalhos de Comando
```
Tecla	Função
INSERT	Ativa/Desativa o movimento do celular (Só funciona em tela cheia).
Q	Mostra ou Esconde o cursor do mouse (útil para ajustes).
ESC	Fecha o programa de controle Python
```

    
