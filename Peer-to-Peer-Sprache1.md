Webseite erstellen mit eigenem vollständige Peer-to-Peer-Sprachkommunikation über WebRTC?

Um eine vollständige Peer-to-Peer-Sprachkommunikation über **WebRTC** auf deiner Webseite zu implementieren, musst du nicht nur das Aufnehmen und Abspielen von Audio (wie im ersten Beispiel), sondern auch den Aufbau einer direkten Verbindung zwischen zwei Benutzern (Peers) ermöglichen. Dazu benötigen wir eine Kombination aus **WebRTC** und einem **Signalisierungsserver**.

### Schritte zur Implementierung einer Peer-to-Peer-Sprachkommunikation mit WebRTC

1. **Client-Seite**: Die Webseite, die WebRTC verwendet, um Audio-Daten zwischen den Benutzern auszutauschen.
2. **Signalisierungsserver**: Der Server, der die Verbindungsdetails (Angebot und Antwort) zwischen den Peers vermittelt.
3. **STUN/TURN-Server**: Diese Server helfen, die Netzwerkverbindungen zwischen den Peers durch Firewalls und NAT zu etablieren.

#### Voraussetzungen:
- **Node.js** für den Signalisierungsserver.
- **WebSockets** zur Übermittlung der Signalisierungsdaten zwischen den Peers.
- **WebRTC API** im Frontend für die Audio-Übertragung.

### 1. **Frontend (Client-Seite)**

Hier ein einfaches Beispiel, wie zwei Benutzer miteinander über WebRTC kommunizieren können. Die Verbindung und der Datenaustausch laufen über WebRTC, während die Verbindungsinformationen über einen WebSocket-basierten Signalisierungsserver gesendet werden.

```html
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebRTC Peer-to-Peer Sprachkommunikation</title>
</head>
<body>
    <h1>WebRTC Sprachkommunikation</h1>
    <button id="startCall">Anruf starten</button>
    <button id="hangupCall" disabled>Auflegen</button>
    <audio id="remoteAudio" controls autoplay></audio>

    <script>
        const startCallButton = document.getElementById('startCall');
        const hangupCallButton = document.getElementById('hangupCall');
        const remoteAudio = document.getElementById('remoteAudio');

        let localStream;
        let peerConnection;
        const signalingServer = new WebSocket('ws://localhost:3000'); // Verbinde mit dem Signalisierungsserver

        // WebRTC Konfiguration mit STUN-Server
        const configuration = {
            iceServers: [
                { urls: 'stun:stun.l.google.com:19302' } // Google STUN-Server
            ]
        };

        // WebSocket-Nachricht empfangen
        signalingServer.onmessage = async (message) => {
            const data = JSON.parse(message.data);

            if (data.offer) {
                // Empfangene Angebot (Offer) verarbeiten
                await peerConnection.setRemoteDescription(new RTCSessionDescription(data.offer));
                const answer = await peerConnection.createAnswer();
                await peerConnection.setLocalDescription(answer);
                signalingServer.send(JSON.stringify({ answer }));
            } else if (data.answer) {
                // Empfangene Antwort (Answer) verarbeiten
                await peerConnection.setRemoteDescription(new RTCSessionDescription(data.answer));
            } else if (data.iceCandidate) {
                // ICE-Kandidat verarbeiten
                try {
                    await peerConnection.addIceCandidate(data.iceCandidate);
                } catch (e) {
                    console.error('Fehler beim Hinzufügen von ICE-Kandidaten:', e);
                }
            }
        };

        // Anruf starten
        startCallButton.addEventListener('click', async () => {
            startCallButton.disabled = true;
            hangupCallButton.disabled = false;

            // Zugriff auf das Mikrofon und Audio streamen
            localStream = await navigator.mediaDevices.getUserMedia({ audio: true });

            // Neue WebRTC-Verbindung aufbauen
            peerConnection = new RTCPeerConnection(configuration);

            // Lokale Audio-Stream dem PeerConnection hinzufügen
            localStream.getTracks().forEach(track => peerConnection.addTrack(track, localStream));

            // Remote-Stream auf der Webseite anzeigen
            peerConnection.ontrack = (event) => {
                remoteAudio.srcObject = event.streams[0];
            };

            // ICE-Kandidaten (Verbindungsdetails) senden
            peerConnection.onicecandidate = (event) => {
                if (event.candidate) {
                    signalingServer.send(JSON.stringify({ iceCandidate: event.candidate }));
                }
            };

            // Ein Angebot (Offer) erstellen und senden
            const offer = await peerConnection.createOffer();
            await peerConnection.setLocalDescription(offer);
            signalingServer.send(JSON.stringify({ offer }));
        });

        // Auflegen
        hangupCallButton.addEventListener('click', () => {
            peerConnection.close();
            hangupCallButton.disabled = true;
            startCallButton.disabled = false;
        });
    </script>
</body>
</html>
```

### 2. **Signalisierungsserver (Backend)**

Ein einfacher Signalisierungsserver mit **WebSocket**. Dieser vermittelt nur die Verbindungsinformationen (Angebot, Antwort und ICE-Kandidaten) zwischen den Peers.

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 3000 });

wss.on('connection', (ws) => {
    ws.on('message', (message) => {
        // Nachricht von einem Peer empfangen und an den anderen weiterleiten
        wss.clients.forEach(client => {
            if (client !== ws && client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    });

    ws.on('close', () => {
        console.log('Verbindung geschlossen');
    });
});

console.log('Signalisierungsserver läuft auf ws://localhost:3000');
```

### 3. **Erklärung der Funktionsweise**

1. **getUserMedia()**: Greift auf das Mikrofon zu und erzeugt einen lokalen Audio-Stream.
2. **RTCPeerConnection**: Diese WebRTC-API wird verwendet, um die Peer-Verbindung zu erstellen. Die Verbindung verwendet ICE-Kandidaten, um das Netzwerk zu durchdringen (z. B. hinter einem NAT-Router).
3. **Signalisierungsserver**: Ein einfacher WebSocket-Server, der als Vermittler fungiert und Verbindungsdaten wie Angebot, Antwort und ICE-Kandidaten zwischen den Peers austauscht.
4. **STUN-Server**: WebRTC verwendet STUN-Server, um die externe IP-Adresse und Ports für die Verbindung zu ermitteln. Im Beispiel wird der Google STUN-Server verwendet (`stun:stun.l.google.com:19302`).

### 4. **Ablauf**

1. **Signalisierung**: Ein Peer erstellt ein **Offer** und sendet es über den Signalisierungsserver an den anderen Peer.
2. **Verbindungsaufbau**: Der zweite Peer empfängt das Offer, erstellt eine **Answer** und sendet sie zurück.
3. **ICE-Kandidaten**: Beide Peers tauschen ICE-Kandidaten aus, um die beste Verbindung herzustellen.
4. **Audioübertragung**: Sobald die Verbindung steht, werden die Audio-Daten direkt zwischen den Peers übertragen, ohne dass sie über den Server laufen.

### 5. **Erweiterungen**
- **TURN-Server**: Für Verbindungen hinter strikten Firewalls oder NATs könnte ein **TURN-Server** erforderlich sein, der als Relay fungiert, falls eine direkte Verbindung zwischen den Peers nicht möglich ist.
- **Benutzeroberfläche**: Du kannst die Benutzeroberfläche mit weiteren Funktionen wie Anrufliste, Videoübertragung oder einer erweiterten Fehlerbehandlung anpassen.

### Fazit

Dieses Setup ermöglicht die direkte Peer-to-Peer-Sprachkommunikation zwischen zwei Benutzern über WebRTC. Der Signalisierungsserver übermittelt nur die Verbindungsinformationen, und die eigentlichen Audio-Daten werden direkt zwischen den Peers ausgetauscht.