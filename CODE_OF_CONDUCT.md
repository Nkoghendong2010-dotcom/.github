const { default: makeWASocket, useMultiFileAuthState, DisconnectReason, useSingleFileAuthState } = require('@whiskeysockets/baileys')
const readline = require("readline")

const rl = readline.createInterface({ input: process.stdin, output: process.stdout })
const question = (text) => new Promise((resolve) => rl.question(text, resolve))

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth')

    const sock = makeWASocket({
        auth: state,
        printQRInTerminal: false // On désactive le QR
    })

    sock.ev.on('creds.update', saveCreds)

    // Si pas connecté, on demande le numéro pour générer le code
    if (!sock.authState.creds.registered) {
        const phoneNumber = await question('Entre ton numéro WhatsApp avec l’indicatif, ex: 33612345678\n> ')
        const code = await sock.requestPairingCode(phoneNumber)
        console.log(`\nCODE D'APPAIRAGE: ${code}\n`)
        console.log(`Va dans WhatsApp > Paramètres > Appareils connectés > Lier un appareil > Lier avec le numéro de téléphone`)
        rl.close()
    }

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update

        if(connection === 'close') {
            const shouldReconnect = lastDisconnect?.error?.output?.statusCode!== DisconnectReason.loggedOut
            console.log('Déconnecté. Reconnexion:', shouldReconnect)
            if(shouldReconnect) startBot()
        }
        else if(connection === 'open') {
            console.log('✅ Bot connecté à WhatsApp!')
        }
    })

    sock.ev.on('messages.upsert', async ({ messages }) => {
        const msg = messages[0]
        if (!msg.message || msg.key.fromMe) return

        const text = msg.message.conversation || msg.message.extendedTextMessage?.text
        const sender = msg.key.remoteJid

        if (text === '!ping') {
            await sock.sendMessage(sender, { text: 'Pong 🏓' })
        }
        else if (text === '!menu') {
            await sock.sendMessage(sender, {
                text: `*Menu du bot*\n\n!ping - Test\n!info - Infos bot`
            })
        }
    })
}

startBot()
