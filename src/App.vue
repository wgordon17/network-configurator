<template>
  <div id="app" class="container">
    <div class="row">
      <form class="col-md-4">
        <div class="form-group">
          <h3>Availability Zone Config</h3>
          <div class="form-check">
            <input
              @input="e => azConfig = 'single'"
              type="radio"
              name="azConfig"
              value="single"
              id="single-az"
              class="form-check-input"
              checked
            >
            <label for="single-az" class="form-check-label">Single-AZ</label>
          </div>
          <div class="form-check">
            <input
              @input="e => azConfig = 'multi'"
              type="radio"
              name="azConfig"
              value="multi"
              id="multi-az"
              class="form-check-input"
            >
            <label for="multi-az" class="form-check-label">Multi-AZ</label>
          </div>
        </div>
        <div class="form-group mt-5">
          <h3>Network Config</h3>
          <label for="machine">Machine CIDR</label>
          <input
            @input="e => machineCIDR = e.target.value || initial['machineCIDR']"
            type="text"
            name="machine"
            class="form-control form-control-lg"
            :placeholder="machineCIDR"
          >
          <div v-show="!validCidr(machineCIDR)" class="invalid-text">
            Use proper CIDR notation, e.g., {{ initial['machineCIDR'] }}
          </div>
          <label for="service">Service CIDR</label>
          <input
            @input="e => serviceCIDR = e.target.value || initial['serviceCIDR']"
            type="text"
            name="service"
            class="form-control form-control-lg"
            :placeholder="serviceCIDR"
          >
          <div v-show="!validCidr(serviceCIDR)" class="invalid-text">
            Use proper CIDR notation, e.g., {{ initial['serviceCIDR'] }}
          </div>
          <label for="pod">Pod CIDR</label>
          <input
            @input="e => podCIDR = e.target.value || initial['podCIDR']"
            type="text"
            name="pod"
            class="form-control form-control-lg"
            :placeholder="podCIDR"
          >
          <div v-show="!validCidr(podCIDR)" class="invalid-text">
            Use proper CIDR notation, e.g., {{ initial['podCIDR'] }}
          </div>
          <label for="prefix">Host Prefix</label>
          <input
            @input="e => hostPrefix = e.target.value || initial['hostPrefix']"
            type="text"
            name="prefix"
            class="form-control form-control-lg"
            :placeholder="hostPrefix"
          >
          <div v-show="!validPrefix(hostPrefix)" class="invalid-text">
            Use proper host subnet notation, e.g., {{ initial['hostPrefix'] }}
          </div>
        </div>
      </form>
      <div class="col-md-6 my-auto">
        <p v-if="totalNodeCount">
          Maximum number of possible nodes: {{ totalNodeCount }}<br>
          <span :class="[ hasEnoughNodes ? 'valid-text': 'invalid-text' ]">(Minimum of {{ nodeMinimum }} nodes required)</span><br>
          <span style="font-size: 80%">Node count limited by {{ nodeLimitations }}</span>
        </p>
        <p v-if="serviceCount">
          Maximum number of possible services in a cluster: {{ serviceCount }}<br>
          <span v-if="serviceCount < 5000" class="invalid-text">Having a limited Service CIDR can severely limit your ability to scale in the future.</span>
        </p>
        <p v-if="podCount">
          Maximum number of possible pods on a node: {{ podCount }}
        </p>
      </div>
    </div>
  </div>
</template>

<script>
let Netmask = require('netmask').Netmask;
let cidrRegex = /^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\/[0-9]{1,2}$/;
let prefixRegex = /^\/[0-9]{1,2}$/;

export default {
  name: 'App',
  data() {
    return {
      machineCIDR: '10.0.0.0/22',
      serviceCIDR: '172.30.0.0/16',
      podCIDR: '10.128.0.0/20',
      hostPrefix: '/23',
      azConfig: 'single',
      initial: {}
    }
  },
  created() {
    this.initial['machineCIDR'] = this.machineCIDR;
    this.initial['serviceCIDR'] = this.serviceCIDR;
    this.initial['podCIDR'] = this.podCIDR;
    this.initial['hostPrefix'] = this.hostPrefix;
  },
  methods: {
    validCidr(cidr) {
      return cidrRegex.test(cidr);
    },
    validPrefix(prefix) {
      return prefixRegex.test(prefix);
    }
  },
  computed: {
    nodeMinimum() {
      return this.azConfig === 'single' ? 10 : 15;
    },
    machineNetmask() {
      if (this.validCidr(this.machineCIDR)) {
        let machine_subnets = this.machineCIDR.split('/');
        // machine subnets need 4 added to /<whatever> to account for being split into 16 subnets
        return new Netmask(machine_subnets[0] + '/' + (parseFloat(machine_subnets[1]) + 4));
      } else {
        return false;
      }
    },
    serviceNetmask() {
      return this.validCidr(this.serviceCIDR) ? new Netmask(this.serviceCIDR): false;
    },
    podNetmask() {
      return this.validCidr(this.podCIDR) ? new Netmask(this.podCIDR) : false;
    },
    podSlash() {
      return this.podNetmask ? this.podCIDR.split('/')[1] : false;
    },
    prefix() {
      return this.validPrefix(this.hostPrefix) ? this.hostPrefix.split('/')[1] : false;
    },
    hasEnoughNodes() {
      return this.totalNodeCount >= this.nodeMinimum;
    },
    machineNodeCount() {
      if (this.machineNetmask) {
        let size = this.machineNetmask.size;
        // netmask size needs to be reduced by 9 for each subnet
        //       to account for AWS reserved IP's as well as ELB instances
        let nodeCount = size - 9;
        // netmask size is multiplied by 3 for multi-az (3 subnets available)
        if (this.azConfig === 'multi') {
          nodeCount *= 3;
        }
        return nodeCount;
      } else {
        return false;
      }
    },
    podNodeCount() {
      return (this.prefix && this.podSlash) ? 2 ** (parseFloat(this.prefix) - parseFloat(this.podSlash)) : false;
    },
    totalNodeCount() {
      if (this.machineNodeCount && this.podNodeCount) {
        return Math.min(this.machineNodeCount, this.podNodeCount);
      } else if (this.machineNodeCount) {
        return this.machineNodeCount;
      } else if (this.podNodeCount) {
        return this.podNodeCount;
      } else {
        return false;
      }
    },
    nodeLimitations() {
      if (this.machineNodeCount && this.podNodeCount) {
        if (this.machineNodeCount > this.podNodeCount) {
          return "Pod CIDR";
        } else {
          return "Machine CIDR";
        }
      } else if (this.machineNodeCount) {
        return "Machine CIDR";
      } else if (this.podNodeCount) {
        return "Pod CIDR";
      } else {
        return false;
      }
    },
    serviceCount() {
      if (this.serviceNetmask) {
        return this.serviceNetmask.size;
      } else {
        return false;
      }
    },
    podCount() {
      if (this.prefix) {
        // reduce pods/node by 3 to account for node reservations
        return (2 ** (32 - parseFloat(this.prefix))) - 3;
      } else {
        return false;
      }
    }
  }
};
</script>

<style lang="sass">
@import '../node_modules/bootstrap/scss/bootstrap'
@import '../node_modules/bootstrap-vue/src/index.scss'

#app
  font-family: Avenir, Helvetica, Arial, sans-serif
  -webkit-font-smoothing: antialiased
  -moz-osx-font-smoothing: grayscale
  text-align: center
  margin-top: 60px
  color: #2c3e50

.form-group
  text-align: left
  color: black

.invalid-text
  width: 100%
  margin-top: 0.25rem
  font-size: 80%
  color: #dc3545

.valid-text
  width: 100%
  margin-top: 0.25rem
  font-size: 80%
  color: #007915
</style>
