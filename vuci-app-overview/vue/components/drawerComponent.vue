<template>
  <div>
    <div v-if="buttonVisibility" class="abutton" @click="toggleVisibility">
        <a-icon class="icon" type="setting" />
    </div>
    <a-drawer :visible="isVisible" title="Settings" width="350" :closable="false" :footer-style="{ textAlign: 'left' }" @close="toggleVisibility">
        <div class="button" @click="toggleVisibility">
            <a-icon class="icon" type="setting" />
        </div>
        <a-checkbox-group @change="checkboxChange" v-model="selectedCards" :options="cardsOptions" :style="{'display':'grid'}"/>
    </a-drawer>
  </div>
</template>
<script>
export default ({
  props: {
    cards: {
      type: Array,
      required: true
    }
  },
  data () {
    return {
      isVisible: false,
      buttonVisibility: true,
      selectedCards: []
    }
  },
  methods: {
    async  checkboxChange (self) {
      await this.$uci.load('overview')
      const checkSid = await this.$uci.sections('overview')
      if (Object.keys(checkSid).length === 0) {
        const sid = this.$uci.add('overview', 'cards')
        this.$uci.set('overview', sid, 'visibleCards', self)
      } else {
        const existSid = this.$uci.sections('overview', 'cards')
        this.$uci.set('overview', existSid[0]['.name'], 'visibleCards', self)
      }
      this.$uci.save()
      this.$uci.apply()
      this.$emit('cardsSettings', self)
    },
    toggleVisibility () {
      this.isVisible = !this.isVisible
      this.buttonVisibility = false
    }
  },
  computed: {
    cardsOptions () {
      const options = this.cards.map((card) => ({
        label: card.title.toUpperCase(),
        value: card.title
      }))
      options.sort(function (a, b) {
        var textA = a.label.toUpperCase()
        var textB = b.value.toUpperCase()
        return (textA < textB) ? -1 : (textA > textB) ? 1 : 0
      })
      return options
    }
  },
  async created () {
    await this.$uci.load('overview')
    const sid = await this.$uci.sections('overview', 'cards')
    this.selectedCards = await sid[0].visibleCards
    this.$emit('cardsSettings', this.selectedCards)
  }
})
</script>
<style>
.button {
  height: 50px;
  width: 40px;
  background-color: #59a3f1;
  top: 85px;
  right: 350px;
  border-radius: 8px;
  align-items: center;
  justify-content: center;
  position: fixed;
  display: flex;
}
.abutton {
  height: 50px;
  width: 40px;
  background-color: #59a3f1;
  top: 85px;
  right: 1px;
  border-radius: 8px;
  align-items: center;
  justify-content: center;
  position: fixed;
  display: flex;
}
.icon {
  font-size: 18px;
  color:white
}
</style>
