<template>
  <div class="home">
    <div v-for="facet in facets">
      <div v-if="facet.metadata">
        <h1>{{facet.name}}</h1>
        
        <input type="text" :disabled="facet.metadata.output.devdoc.methods['getCounter(uint256)']['custom:test'] === 'test123'">
      </div>
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
import HelloWorld from '@/components/HelloWorld.vue'
import * as IPFS from 'ipfs-core'
import { ethers } from "ethers";
import { decode as cborDecode } from 'cbor-x';
import bs58 from 'bs58'

export default {
  name: 'Home',
  data() {
    return {
      ipfs: null,
      facets: [
        {
          address: '0x2f3efA6bbDC5fAf4dC1a600765c7B7829e47bE10',
          name: 'LocalFacet',
        }
      ]
    }
  },
  async mounted() {
    this.ipfs = await IPFS.create()

    const provider = new ethers.providers.Web3Provider(window.ethereum)
    await provider.send("eth_requestAccounts", []);
    // const signer = provider.getSigner()

    this.facets[0].metadata = await this.getMetadataFromAddress(this.facets[0].address)
  },
  methods: {
    async getMetadataFromAddress(address) {
      const url = "http://localhost:8545";
      const provider = new ethers.providers.JsonRpcProvider(url);

      const fromHexString = hexString => new Uint8Array(hexString.match(/.{1,2}/g).map(byte => parseInt(byte, 16)));

      const bytecode = await provider.getCode(address);

      const ipfsHashLength = parseInt(`${bytecode.substr(bytecode.length - 4)}`, 16);
      const cborEncoded = bytecode.substring(bytecode.length - 4 - ipfsHashLength * 2, bytecode.length - 4)
      const contractMetadata = cborDecode(fromHexString(cborEncoded))

      const toHexString = bytes => bytes.reduce((str, byte) => str + byte.toString(16).padStart(2, '0'), '');

      console.log(bs58.encode(contractMetadata.ipfs))
      const stream = this.ipfs.cat(bs58.encode(contractMetadata.ipfs))

      let chunks = []

      for await (const chunk of stream) {
        chunks.push(chunk);
      }

      function concat(views) {
        let length = 0
        for (const v of views)
          length += v.byteLength

        let buf = new Uint8Array(length)
        let offset = 0
        for (const v of views) {
          const uint8view = new Uint8Array(v.buffer, v.byteOffset, v.byteLength)
          buf.set(uint8view, offset)
          offset += uint8view.byteLength
        }

        return buf
      }

      const data = concat(chunks)
      return JSON.parse(new TextDecoder().decode(data).toString());
    }
  },
  components: {
    HelloWorld
  }
}
</script>
