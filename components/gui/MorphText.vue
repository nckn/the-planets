<template lang="pug">
  //- h1 Test
  .men__item(rel="nofollow" target="_blank")
    svg.menu__text(viewbox='0 0 110 20' preserveaspectratio='xMinYMid meet')
      defs
        filter#goo-4
          fegaussianblur(in='SourceGraphic' stddeviation='1' result='blur')
          fecolormatrix(in='blur' mode='matrix' values='	1 0 0 0 0  \
          0 1 0 0 0  \
          1 0 1 0 0  \
          0 0 0 15 -8' result='goo')
          fecomposite(in='SourceGraphic' in2='goo' operator='atop')
      g
        text(x='0' y='15') Marley
        text(x='0' y='15') 1981-5-11
</template>

<script>

import { gsap } from 'gsap';
var WebFont = require('webfontloader')

// import blurMorph from '@/assets/js/blur-morph.js'
class Menu {
  constructor(el) {
    this.DOM = {el: el};
    this.items = [];
    // [...this.DOM.el.querySelectorAll('.men__item')].forEach(item => this.items.push(new MenuItem(item)));
  }
}

class MenuItem {
  constructor(el) {
    this.DOM = {el: el};
    this.DOM.textsGroupEl = this.DOM.el.querySelector('svg > g');
    this.filterId = this.DOM.el.querySelector('svg filter').id;
    
    [this.DOM.text_1, this.DOM.text_2] = this.DOM.textsGroupEl.querySelectorAll('text');
    
    this.DOM.feBlur = document.querySelector(`#${this.filterId} > feGaussianBlur`);
    this.primitiveValues = {stdDeviation: 0};

    this.createTimeline();
    this.initEvents();
  }
  initEvents() {
    this.onMouseEnterFn = () => {
      this.DOM.textsGroupEl.style.filter = `url(#${this.filterId})`;
      this.tl.play();
      console.log('should play')
      console.log(this.DOM.text_1)
    }
    this.onMouseLeaveFn = () => {
      this.DOM.textsGroupEl.style.filter = `url(#${this.filterId})`;
      this.tl.reverse();
      console.log('should leave')
    }
    this.DOM.el.addEventListener('mouseenter', this.onMouseEnterFn);
    this.DOM.el.addEventListener('mouseleave', this.onMouseLeaveFn);
  }
  createTimeline() {
    // init timeline
    this.tl = gsap.timeline({
      paused: true,
      onComplete: () => {
        this.DOM.textsGroupEl.style.filter = 'none';
      },
      onReverseComplete: () => {
        this.DOM.textsGroupEl.style.filter = 'none';
      },
      onUpdate: () => {
        this.DOM.feBlur.setAttribute('stdDeviation', this.primitiveValues.stdDeviation)
      }
    })
    
    .to(this.primitiveValues, { 
      duration: 0.8,
      ease: "none",
      startAt: {stdDeviation: 0},
      stdDeviation: 1
    }, 0)
    .to(this.primitiveValues, { 
      duration: 0.8,
      ease: "none",
      stdDeviation: 0
    })
    .to(this.DOM.text_1, { 
      duration: 1.6,
      ease: "none", // Power1.easeInOut
      opacity: 0
    }, 0)
    .to(this.DOM.text_2, { 
      duration: 1.6,
      ease: "none", // Power1.easeInOut
      opacity: 1
    }, 0);
  }
}

export default {
  name: "MorphText",
  props: [''],
  data() {
    return {
      active: false,
      menu: null
    };
  },
  mounted() {
    var self = this;
    // var morphCode = require('@/assets/js/blur-morph.js')
    // console.log('menu')
    // console.log(morphCode)
    // self.setupAudioContext()
    // console.log('heres a cell')
    // console.log(this.isgreen)
    self.$nextTick(
      self.setupMorph()
    )
  },
  methods: {
    setupMorph() {
      var self = this
      // Preload images
      const preloadFonts = (id) => {
        return new Promise((resolve, reject) => {
          WebFont.load({
            typekit: {
              id: id
            },
            active: resolve
          });
        });
      };

      // Preload typekit fonts
      preloadFonts('dba6omz').then(() => {
        // document.body.classList.remove('loading');
        console.log('has loaded alright')
      });

      self.menu = new Menu(document.querySelector('nav.menu'));

      console.log(self.menu)
      console.log(...document.querySelectorAll('.men__item'))

      document.querySelectorAll('.men__item').forEach(item => self.menu.items.push(new MenuItem(item)));
      // [...self.menu.DOM.el.querySelectorAll('.men__item')].forEach(item => self.menu.items.push(new MenuItem(item)));
    }
  },
};
</script>

<style lang="scss" scoped>

/* Page Loader */
// .js .loading::before,
// .js .loading::after {
//     content: '';
//     position: fixed;
//     z-index: 1000;
// }

// .js .loading::before {
//     top: 0;
//     left: 0;
//     width: 100%;
//     height: 100%;
//     background: var(--color-bg);
// }

// .js .loading::after {
//     top: 50%;
//     left: 50%;
//     width: 60px;
//     height: 60px;
//     margin: -30px 0 0 -30px;
//     border-radius: 50%;
//     opacity: 0.4;
//     background: var(--color-link);
//     animation: loaderAnim 0.7s linear infinite alternate forwards;

// }

// @keyframes loaderAnim {
//     to {
//         opacity: 1;
//         transform: scale3d(0.5, 0.5, 1);
//     }
// }

.hidden {
	opacity: 0;
	position: absolute;
	pointer-events: none;
	z-index: -1;
}

$color-link: #ff00ff;
$color-link-hover: #ff0000;

a {
	text-decoration: none;
	color: $color-link;
	outline: none;
	border-bottom: 2px solid;
	transition: border-color 0.1s;
}

a:hover,
a:focus {
	color: $color-link-hover;
	outline: none;
	border-color: transparent;
}

main {
	padding: 3rem 5vw;
	position: relative;
	z-index: 1000;
}

.title {
	font-size: 1rem;
	margin: 0 0 1rem;
	font-weight: normal;
}

.links {
	display: inline;
}

.demos {
	margin: 1rem 0;
}

.demo--current,
.demo--current:hover {
	border-color: transparent;
}

.credits a {
	color: currentColor;
}

.menu {
	width: 100%;
	min-height: 400px;
	display: grid;
}

.men__item {
  border: none;
  transition: none;
  cursor: pointer;
  position: relative;
  width: 100px;
  height: 100px;
  pointer-events: all;
}

$color-menu: #808080;
$font-menu: sans-serif;

.menu__text {
	font-size: 16px;
	fill: $color-menu;
  display: block;
  font-family: $font-menu;
	font-weight: 700;
  position: absolute;
  height: 120%;
  width: 100%;
  pointer-events: none;
}

.menu__text text {
	transform-origin: 50% 50%;
	transform-box: view-box;
}

.menu__text text:nth-child(2) { 
	opacity: 0;
}

</style>

