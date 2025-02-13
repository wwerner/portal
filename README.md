# Example

```js
import * as http from 'http'
import * as crypto from 'node:crypto'
import { TLSSocket } from 'tls'
import { Transform } from 'node:stream'
import { createServer } from './Portal.js'

const PORT = 1337
const HOST = '127.0.0.1'

const proxy = createServer({
  requestTransformer: (request) => new Transform({
    transform: (chunk, _encoding, callback) => {
      console.log('Raw data to be passed in the request', chunk.toString())
      callback(null, chunk)
    }
  }),
  responseTransformer: (response, request) => new Transform({
    transform: (chunk, _encoding, callback) => {
      console.log('Raw data to be passed in the response', chunk.toString())
      callback(null, chunk)
    }
  }),
  clientOptions: async (request) => {
    return {} // a custom key and cert could be returned here
  },
  serverOptions: async (request) => {
    return {
      // This flag allows legacy insecure renegotiation between OpenSSL and unpatched servers
      // @see {@link https://stackoverflow.com/questions/74324019/allow-legacy-renegotiation-for-nodejs}
      secureOptions: crypto.constants.SSL_OP_LEGACY_SERVER_CONNECT
    }
  }
})

proxy.on('request', (request) => {
  console.log('Parsed request to observe', request.headers)
})

proxy.on('response', (response, request) => {
  console.log('Parsed response to observe', response.headers)
})

proxy.listen(PORT, HOST)

/*
 * Make an example request
 */
proxy.on('listening', () => {
  const options = {
    port: PORT,
    host: HOST,
    method: 'CONNECT',
    path: 'example.com:443'
  }

  const req = http.request(options)
  req.end()

  req.on('connect', (res, socket, head) => {
    const upgradedSocket = new TLSSocket(socket, {
      rejectUnauthorized: false,
      requestCert: false,
      isServer: false
    })

    upgradedSocket.write('GET / HTTP/1.1\r\n' +
      'Host: example.com:443\r\n' +
      'Connection: close\r\n' +
      '\r\n')
  })
})

```
