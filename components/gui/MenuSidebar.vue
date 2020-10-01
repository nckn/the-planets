<template lang="pug">
  .menu-container(ref="menu_container" v-bind:class="style_name" name="menu-container")
    .menu-trigger(v-if="type === 'hover'" @mouseenter="hoverOption" @touchstart="hoverOption" name="menu-trigger" v-bind:class="{ istiny: revealMenu }")
      svg.hamburger(width='24' height='24' viewbox='0 0 24 24' fill='none' xmlns='http://www.w3.org/2000/svg')
        g(fill='none' stroke='#b0b0b0' stroke-width='1' stroke-linecap='round')
          path(d='M5.5 11.5L19.5 11.5')
          path(d='M9.5 5.5L12.5 5.5')
          path(d='M9.5 17.5L12.5 17.5')
          path(d='M12.5 5.5L15.5 5.5')
          path(d='M12.5 17.5L15.5 17.5')
    .menu-trigger(v-else-if="type === 'info'" @click="hoverOption" @touchstart="hoverOption" name="menu-trigger" v-bind:class="[{ istiny: revealMenu }, type ]")
      .icon
    .menu-options-container(v-if="type === 'hover'" v-bind:class="{ isin: revealMenu }")
      .menu-option-group(v-for="(pm, index) in options" :name="pm.name" :key="`pm-${index}`" v-bind:class="`pie${index+1}`")
        .menu-option(v-for="(mtd, index) in pm.methods" :key="`mtd-${index}`" @click="commandClicked" :name="`${mtd.name}`" v-bind:class="[`${pm.name} ${mtd.name}`, {disabled: mtd.disabled}]" disabled="mtd.disabled")
    .menu-options-container.menu-info(v-else-if="type === 'info'" v-bind:class="{ isin: revealMenu }")
      h1 The Silent Web
      p.grey-it A time for peace and quiet.
      p.description.small-it Click, zoom, explore, pan and most of all – Listen. Disappear for a while.
</template>

<script>

import globalFunctions from '@/mixins/globalFunctions.js'
// import { radians, map, distance } from '@/assets/js/helpers'

// Based on
// https://codepen.io/ainalem/pen/YoyZpq

export default {
  name: 'MenuOptions',
  props: ['type', 'options', 'closer', 'style_name'],
  mixins: [globalFunctions],
  components: {
    //
  },
  data () {
    return {
      menuContainer: null,
      revealMenu: false,
      revealMenuSub: false,
      curName: '',
      // delay between two menus
      delay: 100,
      isTouch: (('ontouchstart' in window) || (navigator.MaxTouchPoints > 0) || (navigator.msMaxTouchPoints > 0)),
      closerEvent: null,
      // Debug
      debug: false
    }
  },
  watch: {
    closer () {
      console.log('this thing: ', this.closer)
      // console.log('the renderer: ', self.$root)
      // Needed to make the menu disappear again
      if (this.debug) {
        return
      }
      this.setupCloser()
    }
  },
  async mounted() {
    var self = this

    // alert(self.isTouch)

    // Set menu container
    self.menuContainer = self.$refs.menu_container
    // await console.log('this is the closer: ', this.closer)

    // See if touch or mouse
    // self.closerEvent = self.isTouch ? 'touchmove' : 'closerEvent'
    // self.closerEvent = self.isTouch ? 'touchstart' : 'mouseenter'
    // self.closerEvent = 'click'

    // setTimeout(() => {
    //   console.log('this is the closer: ', this.closer)
    //   // self.init()
    // }, 1000)
    // console.log('length: ', this.options.length)
  },
  methods: {
    commandClicked(e) {
      var self = this
      var target = e.target || e.srcElement
      var name = target.getAttribute('name')
      var disabled = target.getAttribute('disabled')
      var obj = {target: target, name: name}
      self.$parent.clickedCommandInChild(obj)
      // if (name === 'sun') {
        //   target.classList.add('off')
      // }
      // target.setAttribute('disabled', false)
      if (target.classList.contains('off')) {
        // console.log('target: ', disabled)
        target.classList.remove('off')
        // target.setAttribute('disabled', false)
      }
      else {
        target.classList.add('off')
        // target.disabled = true
        // target.setAttribute('disabled', true)
      }
    },
    hideMenuAgain(e) {
      var self = this
      var target = e.target || e.srcElement
      var name = target.getAttribute('name')
      // console.log('this is it: ', e)
      self.revealMenu = false
    },
    // Setup menu closer
    setupCloser () {
      var self = this
      self.closer.addEventListener('touchstart', (e) => {
        self.hideMenuAgain(e)
      }, true);
      self.closer.addEventListener('touchmove', (e) => {
        self.hideMenuAgain(e)
      }, true);
      self.closer.addEventListener('mouseenter', (e) => {
        self.hideMenuAgain(e)
      }, true);
    },
    // rndmColor () {
    //   return this.randomColor()
    // },
    init() {
      var self = this
      window.addEventListener( 'resize', self.onWindowResize(), false )
      // window.addEventListener( 'keydown', function ( event ) {
      //   switch ( event.keyCode ) {
      //     case 81: // Q
      //       // self.control.setSpace( self.control.space === 'local' ? 'world' : 'local' )
      //       break
      //   }
      // })

      // window.addEventListener( 'keyup', function ( event ) {
      //   switch ( event.keyCode ) {
      //     case 16: // Shift
      //       self.control.setTranslationSnap( null )
      //       self.control.setRotationSnap( null )
      //       self.control.setScaleSnap( null )
      //       break
      //   }
      // })
    },
    hoverOption(e) {
      var self = this
      var target = e.target || e.srcElement
      var name = target.getAttribute('name')

      console.log('menu has been triggered')

      self.revealMenu = true
      // if (name === 'menu-trigger') {
      //   self.revealMenu = true
      // }
      
      // console.log('target: ', self.curName)
      if (e.type === 'mouseenter' && !self.revealMenuSub) {
        setTimeout(() => {
          self.revealMenuSub = true
        }, self.delay)
        // self.curName = name
        // Apply temporary eventlistener
        // document.addEventListener('mouseenter', function clearMouseUp(e) {
        //   var name = target.getAttribute('name')
        //   console.log('this is it: ', name)
        //   if (name === 'container-tfctls') {
        //     self.revealMenuSub = false
        //     document.removeEventListener('mouseenter', clearMouseUp, true);
        //   }
        // }, true);
      }
      // if (e.type === 'mouseleave' && self.curName != name) {
      //   // self.curName = name
      //   // self.revealMenuSub = false
      //   // self.curName = name
      // }
      // console.log('name: ', name)
      // if (name === 'menu-container') {
      //   self.revealMenuSub = false
      // }
      // if (self.revealMenuSub) {
      //   console.log('revealed yes: ', e.type)
      //   target.addEventListener('mouseleave', e => {
      //     if (self.curName === name) {
      //       console.log('same name')
      //       self.curName === name
      //       return
      //     }
      //     else {
      //       console.log('not the same name')
      //       self.revealMenuSub = false
      //     } 
      //   })
      // }
    }
  }
};
</script>

<style lang="scss" scoped>

$menu-w: 48px;
$info-w: 300px;

$trigger-s: 48px;
$std-time: 0.25s;

@mixin makeBackground($url, $size) {
  background: url($url) no-repeat;
  background-size: $size $size;
  background-position: center;
}

// Debug
// .menu-trigger { background: red; }
// .menu-options-container { background: blue; }
// .menu-option { background: green; }

.menu-container {
  width: $menu-w;
  height: 100%;
  position: absolute;
  top: 0;
  z-index: 2;
  &.leftside {
    left: 0;
    .menu-options-container {
      transform: translateX(-$menu-w);
      // &.isin {
      //   transform: translateX(0);
      // }
    }
  }
  &.rightside {
    right: 0;
    .menu-options-container {
      transform: translateX($menu-w);
      // &.isin {
      //   transform: translateX(0);
      // }
    }
  }
  .menu-trigger {
    position: absolute;
    width: $trigger-s;
    height: $trigger-s;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all $std-time ease;
    &.istiny {
      transform: scale(0);
    }
    svg {
      pointer-events: none;
    }
    &.info {
      cursor: pointer;
      .icon {
        @include makeBackground('/icons/icon-info.svg', 24px);
      }
    }
  }
  .menu-options-container {
    width: $menu-w;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    transform: translateX($menu-w);
    transition: all $std-time ease;
    &.isin {
      transform: translateX(0);
      background: black;
    }
    &.menu-info {
      width: $info-w;
      transition: all $std-time*2 ease;
      transform: translateX(-$info-w);
      // pointer-events: none;
      padding: 24px;
      box-sizing: border-box;
      &.isin {
        transform: translateX(0);
        // background: black;
      }
      h2 {
        margin-bottom: 0px;
      }
      .description {
        margin-top: 24px;
      }
      // h2, p {
      //   pointer-events: none;
      // }
    }
    .menu-option {
      width: calc(#{$menu-w} - 4px);
      height: calc(#{$menu-w} - 4px);
      pointer-events: all;
      transition: opacity $std-time ease;
      opacity: 0.65;
      cursor: pointer;
      &:hover {
        opacity: 1;
      }
      &.camerapos {
        &.top {
          @include makeBackground('/icons/icon-camera-top.svg', 24px);
        }
        &.front {
          @include makeBackground('/icons/icon-camera-front.svg', 24px);
        }
        &.center {
          @include makeBackground('/icons/icon-center.svg', 24px);
        }
        &.anim-rotate {
          @include makeBackground('/icons/icon-anim-rotate.svg', 24px);
        }
      }
      &.distribution {
        &.ring {
          @include makeBackground('/icons/icon-distribute-ring-lg.svg', 24px);
        }
        &.ring-sm {
          @include makeBackground('/icons/icon-distribute-ring-sm.svg', 24px);
        }
        &.random {
          @include makeBackground('/icons/icon-distribute-random.svg', 24px);
        }
      }
      &.scale {
        &.scale-actual {
          @include makeBackground('/icons/icon-scale-actual.svg', 24px);
        }
      }
      &.env {
        &.sun {
          @include makeBackground('/icons/icon-sun-switch-on.svg', 24px);
          &.off {
            @include makeBackground('/icons/icon-sun-switch-off.svg', 24px);
          }
        } 
        &.show-info {
          @include makeBackground('/icons/icon-show-info.svg', 24px);
          &.off {
            @include makeBackground('/icons/icon-sun-switch-off.svg', 24px);
          }
        } 
        &.floor {
          @include makeBackground('/icons/icon-floor-on.svg', 24px);
          &.off {
            @include makeBackground('/icons/icon-floor-off.svg', 24px);
          }
        } 
      }
    }
  }
}

</style>
