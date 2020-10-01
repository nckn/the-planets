<template lang="pug">
  .container.outer.black-bg(ref="master_cont")
    #containerAO(ref="physics_cont")
    PageInfo(:name="'Sound Objects'")
    //- MenuOptions(:type="'top-left'" :options="guiControls" :closer="container")
    MenuSidebar(:style_name="'leftside'" :type="'info'" :options="guiControls" :closer="container")
    MenuSidebar(:style_name="'rightside'" :type="'hover'" :options="guiControls" :closer="container")
    //- QuickOptions(:type="'top-left'" :options="quickOptions")
    .annotation(ref="annotation" v-if="showAnnotation")
      p Cube
    .debug-overlay(v-if="debug")
      p.debug-text(ref="debug_text") test
    .play-button(@click="startMusic")
      .play-icon(v-bind:class="{ stop: !isPlaying }")
    //- img.img(src="@/assets/images/basilica/posx.jpg" width="400" height="400")
</template>
 
<script>

// https://codepen.io/eeonk/pen/pxyVrB

import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FirstPersonControls } from 'three/examples/jsm/controls/FirstPersonControls.js'
// import { VRButton } from 'three/examples/jsm/webxr/VRButton.js'

// VR related
// Tiny change
import { VRButton } from 'three/examples/jsm/webxr/VRButton.js'

// Post-processing
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { BloomPass } from 'three/examples/jsm/postprocessing/BloomPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { FilmPass } from 'three/examples/jsm/postprocessing/FilmPass.js';
// import { AfterimagePass } from 'three/examples/jsm/postprocessing/AfterimagePass.js';

// Anti aliasing
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js'
import { FXAAShader } from 'three/examples/jsm/shaders/FXAAShader.js'
import { PixelShader } from 'three/examples/jsm/shaders/PixelShader.js'

const vertexShader = require('@/assets/js/shaders/glow/vertexShader.glsl')
const fragmentShader = require('@/assets/js/shaders/glow/fragmentShader.glsl')

// noise shaders
const nVertex = require('@/assets/js/shaders/vertexShader.glsl')
const nFragment = require('@/assets/js/shaders/fragmentShader.glsl')

// const panoImage = require('@/assets/img/pano.jpg')
const panoImage = require('@/static/svg/white-square.svg')

// import perlinNoise from '@/assets/js/noise/perlin-noise.js'

import PageInfo from '@/components/gui/PageInfo'
// import QuickOptions from '@/components/gui/QuickOptions'
// import MenuOptions from '@/components/gui/MenuOptions'
import MenuSidebar from '@/components/gui/MenuSidebar'

import { map, generateRandomNumber } from '@/assets/js/helpers'

import { TweenMax, Sine, Bounce, Expo } from 'gsap'

import CANNON from 'cannon'

import globalFunctions from '@/mixins/globalFunctions.js'

// Gaze event
// import GazeEvent from 'gaze-event'

// const path = 'audio/fxs/'
// const path = 'snd/';
const path = '';

// const sounds = [
//   'thud-mouth.mp3'
// ]

const roomTone = '';

const sounds = [
  {obj: null, isPlaying: false, audio: null, name: 'Sun', filename: 'snd/sun.mp3', type: 'sun', texture: 'textures/8k_sun.jpg', r: [10, 1]},
  {obj: null, isPlaying: false, audio: null, name: 'Mercury', filename: 'https://ia600609.us.archive.org/19/items/Holst-ThePlanets/Mercurio.mp3', type: 'sphere', texture: 'textures/mercurymap.jpg', r: [1, 1.2]},
  {obj: null, isPlaying: false, audio: null, name: 'Venus', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Venus.mp3', type: 'sphere', texture: 'textures/venusmap.jpg', r: [1, 1.8]},
  {obj: null, isPlaying: false, audio: null, name: 'Earth', filename: 'snd/heartbeat.mp3', type: 'sphere', texture: 'textures/2_no_clouds_4k.jpg', r: [1.2, 1.7]},
  {obj: null, isPlaying: false, audio: null, name: 'Mars', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Marte.mp3', type: 'sphere', texture: 'textures/5672_mars_2k_color.jpg', r: [0.8, 1]},
  {obj: null, isPlaying: false, audio: null, name: 'Jupiter', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Jupiter.mp3', type: 'sphere', texture: 'textures/jupiter_globalmap2.jpg', r: [5, 5.2]},
  {obj: null, isPlaying: false, audio: null, name: 'Saturn', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Saturno.mp3', type: 'sphere', texture: 'textures/saturnmap.jpg', r: [4, 4.2]},
  {obj: null, isPlaying: false, audio: null, name: 'Uranus', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Urano.mp3', type: 'sphere', texture: 'textures/uranusmap.jpg', r: [3.5, 3.7]},
  {obj: null, isPlaying: false, audio: null, name: 'Neptune', filename: 'https://ia800609.us.archive.org/19/items/Holst-ThePlanets/Neptuno.mp3', type: 'sphere', texture: 'textures/neptunemap.jpg', r: [3.5, 3.8]},
  // {obj: null, isPlaying: false, audio: null, name: 'Pluto', filename: 'snd/sitar1-motif-1.mp3', type: 'sphere'},
]

// Should we show the name in 3D text next to sound obj 
const shouldShowText = false;
const shouldShowGuides = false;
const vrEnabled = false;
// const vrEnabled = true;
const scaleVal = 3;
const bS = 1;

const reso = 32;

const stdCamDistance = 200;

const randomPos = false;
const ringSize = 30;

var xElements = 100;
var yElements = 100;
var scale = 25; // how strong is the noise
var zoom = 6;
var seed = 0.015;
var t = 0;
var fps = 30;

// Perlin noise
//var simplex = new SimplexNoise();
var xoff = 0;
var yoff = 0;
var zoff = 0;
var geo;
var direction;
var way;
var newPos;
var perlinMap;
var startPos = new THREE.Vector3(0, 0, 0);

export default {
  name: 'SoundObjects',
  mixins: [globalFunctions],
  components: {
    PageInfo,
    // MenuOptions,
    MenuSidebar,
    // QuickOptions
  },
  data () {
    return {
      debug: false,
      showAnnotation: true,
      guiControls: [
        {name: 'camerapos',
          enabled: true,
          methods: [ 
            {name: 'top', method: 'rotate', path: '/svg/icons/icon-camera-top.svg'},
            {name: 'front', method: 'translate'},
            {name: 'center'},
            {name: 'anim-rotate'},
          ]
        },
        {name: 'distribution',
          enabled: true,
          methods: [
            {name: 'ring', method: 'rotate', path: '/svg/icons/icon-distribute-ring-lg'},
            {name: 'ring-sm', method: 'rotate', path: '/svg/icons/icon-distribute-ring-sm'},
            {name: 'random', method: 'rotate', path: '/svg/icons/icon-distribute-ring-sm'},
          ]
        },
        {name: 'scale',
          enabled: true,
          methods: [
            {name: 'scale-actual'},
          ]
        },
        {name: 'env',
          enabled: true,
          methods: [
            {name: 'sun', disabled: false},
            {name: 'show-info', disabled: false},
            {name: 'floor', disabled: false},
          ]
        },
        // {name: 'togglecam', method: 'toggleCam'},
      ],
      container: null,
      scene: null,
      camera: null,
      renderer: null,
      // Post processing. AO.
      composer: null,
      renderPass: null,
      saoPass: null,
      group: null,
      // CANNON example
      world: null,
      dt: 1 / 60,
      constraintDown: null,
      gplane: null,
      // mesh: null,
      floor: null,
      clickMarker: null,
      controls: null,
      time: Date.now(),
      // projector: null,
      jointBody: null,
      constrainedBody: null,
      mouseConstraint: null,
      mouseDownPos: null,
      // Objects
      sun: null,
      meshes: [],
      bodies: [],
      // cubePos: [],
      maxPos: 180,
      noOFCubes: 4,
      markerMaterial: null,
      lastx: null,
      lasty: null,
      last: null,
      // Audio related
      audioLoader: [],
      analyser1: null,
      listener: null,
      easeTime: 0.15,
      easingType: Sine.easeInOut,
      // controls, FPS related
      clock: null,
      // isWalking: true,
      isWalking: false,
      // Text
      fontLoader: null,
      fonts: [],
      inc: 0,
      INTERSECTED: '',
      intS: null,
      intersectedObject: null,
      // array for sounds to play
      // sounds: [],
      isPlaying: false, // play state
      quickOptions: [
        {name: 'front', path: '/icons/icon-camera-front.svg'},
        {name: 'top', path: '/icons/icon-camera-top.svg'},
        {name: 'angle', path: '/icons/icon-camera-top.svg'},
        // {name: 'snap', path: '/svg/icons/icon-snap-on.svg'},
      ],
      // Post-processing
      params: {
				exposure: 0.6, // org 1
				bloomThreshold: 0.3,
				bloomStrength: 1,
				bloomRadius: 0
      },
      // Passes
      filmPass: null,
      bloomPass: null,
      pixelPass: null,
      fxaaPass: null,
      AfterimagePass: null,
      renderComposer: true,
      // noise
      nMaterial: null,
      start: Date.now(),
      animTime: 0.000025,
      now: null,
      then: Date.now(),
      interval: 1000 / fps,
      delta: null,
      noiseGeometry: null,
      nMaterial: null,
      // Translation
      enableRotation: false,
    }
  },
  mounted () {
    var self = this
    // console.log('CANNON: ', CANNON)
    self.clock = new THREE.Clock()
    // Fonts
    self.fontLoader = new THREE.FontLoader()
    setTimeout(() => {
      self.init();
      // self.initCannon();
      self.createBodies()
      // Orbit controls
      self.setOrbitControls()
      self.render();
    }, 1000)
  },
  // beforeDestroy () {
  //   var self = this
  //   // console.log('before destroy')
  //   window.cancelAnimationFrame( self.reqAnim );
  //   // Dispose controls
  //   self.listener.context.suspend()
  //   self.controls.dispose()
  //   while(self.scene.children.length > 0){
  //     console.log('before destroying: ', self.scene.children[0])
  //     self.scene.remove(self.scene.children[0]);
  //   }
  // },
  methods: {
    init() {
      var self = this
      self.container = self.$refs.physics_cont // Asisgn container
      self.annotation = self.$refs.annotation // Annotation
      
      // console.log('perlinNoise')
      // console.log(noise)

      // scene
      self.scene = new THREE.Scene();
      // self.scene.fog = new THREE.Fog(0x000000, 30, 180) // From SceneControls project
      self.scene.fog = new THREE.Fog(0x000000, 50, 500);

      // camera
      self.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.5, 10000);
      // self.camera.position.set(30, 2, 5)
      self.tweenObject(self.camera.position, 1, {x: 0, y: stdCamDistance, z: 0})
      // self.camera.position.set(0, 2, 0)
      // self.camera.position.set( 0, 2, 50 );
      
      // self.camera.rotation.order = 'YXZ';
      // self.camera.rotation.y = - Math.PI / 4;
      // self.camera.rotation.x = Math.atan( - 1 / Math.sqrt( 2 ) );
      // self.camera.quaternion.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 2);
      self.camera.quaternion.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 2)

      self.scene.add(self.camera);

      // Make audio
      // self.audioLoader = new THREE.AudioLoader();

      self.listener = new THREE.AudioListener()
      // self.listener.context.resume()
      self.camera.add( self.listener )

      // while (self.listener.context.state === 'suspended') {
      //   // console.log(self.listener.context.state)
      // } 

      self.createFloor()

      // Sphere sound
      // var sphere1 = new THREE.SphereBufferGeometry( 20, 32, 16 );
      // var material1 = new THREE.MeshPhongMaterial( { color: 0xff2200, flatShading: true, shininess: 0 } );
      // var mesh1 = new THREE.Mesh( sphere1, material1 );
      // mesh1.position.set( -100, 10, 80 );
      // self.scene.add( mesh1 );

      // var soundTest1 = new THREE.PositionalAudio( self.listener );
      // self.audioLoader.load( path + sounds[0], function ( buffer ) {

      //   soundTest1.setBuffer( buffer );
      //   soundTest1.setRefDistance( 20 );
      //   soundTest1.play();

      // } );
      // mesh1.add( soundTest1 );

      // // Sphere sound 2
      // var sphere2 = new THREE.SphereBufferGeometry( 20, 32, 16 );
      // var material2 = new THREE.MeshPhongMaterial( { color: 0x6622aa, flatShading: true, shininess: 0 } );
      // var mesh2 = new THREE.Mesh( sphere2, material2 );
      // mesh2.position.set( -100, 10, -80 );
      // self.scene.add( mesh2 );

      // var soundTest2 = new THREE.PositionalAudio( self.listener );
      // self.audioLoader.load( path + sounds[3], function ( buffer ) {

      //   soundTest2.setBuffer( buffer );
      //   soundTest2.setRefDistance( 20 );
      //   soundTest2.play();

      // } );
      // mesh2.add( soundTest2 );

      self.renderer = new THREE.WebGLRenderer({
        antialias: true
      });
      self.renderer.setPixelRatio( window.devicePixelRatio );
      self.renderer.setSize(window.innerWidth, window.innerHeight);
      // self.renderer.setClearColor(self.scene.fog.color);

      self.renderer.outputEncoding = THREE.sRGBEncoding;
      self.renderer.xr.enabled = true;
      
      // self.renderer.setClearColor( 0xffffff, 1);

      self.container.appendChild(self.renderer.domElement);

      // self.renderer.gammaInput = true;
      // self.renderer.gammaOutput = true;
      self.renderer.shadowMap.enabled = true;
      // this.renderer.shadowMapSoft = true
      self.renderer.shadowMap.type = THREE.PCFSoftShadowMap; // default THREE.PCFShadowMap
      // this.renderer.shadowMap.type = THREE.PCFSoftShadowMap
      // this.renderer.physicallyCorrectLights = true

      if (shouldShowGuides) {
        self.addGuides()
      }

      self.assignEffects()

      // Add noise ball
      // self.addSun()

      // VR
      if (vrEnabled) {
        self.applyVR();
      }

      // Listen for keys
      self.listenForKeyEvents()

      // Add lights
      self.addSunLight()

      window.addEventListener('touchmove', self.eventHappens, false);
      window.addEventListener('mousemove', self.eventHappens, false);
      
      // window.addEventListener('touchstart', self.eventHappens, false);

      window.addEventListener('touchstart', self.eventHappens, false);
      window.addEventListener('mousedown', self.eventHappens, false);
      
      window.addEventListener('mouseup', self.onMouseUp, false);
      window.addEventListener('touchend', self.onMouseUp, false);

      window.addEventListener( 'resize', self.onWindowResize, false );
    },
    applyVR() {
      var self = this
      self.vrButton = VRButton.createButton( self.renderer )
      document.body.appendChild( self.vrButton );
      self.vrButton.classList.add( 'vrbutton' );

      // Step1: init gazeEvent
      // self.gazeEvent = new GazeEvent();
      // self.renderer.xr.enabled = true;
      // console.log('vr button')
      // console.log(self.vrButton)
    },
    addSunLight() {
      var self = this
      // lights

      // Spot light - start
      // var coneRadius = 2.5
      // var lightAngle = coneRadius / 12
      // var spotLight = new THREE.SpotLight();
      // spotLight.color = 0xffffff;
      // spotLight.position.set( 0, 30, 0);
      // spotLight.exponent = 30
      // // spotLight.angle = lightAngle
      // // spotLight.angle = 0 // Org: Math.PI/3
      // spotLight.intensity = 5
      // // Determine if shown or not on start
      // // spotLight.visible = sceneSettings.checkbox[0].checked
      // // Soften the edge of the light contact
      // spotLight.penumbra = 0.52
      // self.scene.add(spotLight);
      // Spot light - end

      var ambientLight = new THREE.AmbientLight(0x666666);
      // ambientLight.castShadow = true;
      // ambientLight.receiveShadow = true;
      self.scene.add(ambientLight);
      // var light = new THREE.DirectionalLight(0xffffff, 1.75);
      var light = new THREE.PointLight( 0xffa800, 0, 100 );
      light.intensity = 5
      
      var d = 20;
      light.receiveShadow = true;
      light.castShadow = true;
      // light.shadowCameraVisible = true;

      light.shadow.mapSize.width = 2048;
      light.shadow.mapSize.height = 2048;

      light.shadow.camera.left = -d;
      light.shadow.camera.right = d;
      light.shadow.camera.top = d;
      light.shadow.camera.bottom = -d;

      // light.shadow.camera.left = -d;
      // light.shadow.camera.right = d;
      // light.shadow.camera.top = d;
      // light.shadow.camera.bottom = -d;

      // light.shadow.camera.far = 3 * d;
      // light.shadow.camera.near = d;
      // light.shadowDarkness = 0.5;

      self.scene.add(light);
    },
    // addSun() {
    //   var self = this
    //   var radius = 2, segments = 40;
    //   var textureSun = new THREE.TextureLoader().load( '/static/textures/8k_sun.jpg' )
    //   self.sunGeometry = new THREE.SphereGeometry(radius, segments, segments)
    //   self.sunMaterial = new THREE.MeshPhongMaterial({
    //     map: textureSun,
    //     emissive: '#F8CE3B',
    //     specular: new THREE.Color('grey')								
    //   })
    //   this.sunMesh = new THREE.Mesh(this.sunGeometry, this.sunMaterial)
    //   this.scene.add(this.sunMesh)
    // },
    randomIntFromInterval(min, max) { // min and max included 
      return Math.random() * (max - min + 1) + min;
    },
    distributeBodies(layout) {
      var self = this
      var min = -20;
      var max = 10;
      var stdY = bS * 2 // org: bS * 2
      // self.bodies.forEach((body, i) => {
      self.meshes.forEach((body, i) => {
        var x, y, z;
        if (layout === 'random') { 
          x = map(self.randomIntFromInterval(0, 2), 0, 2, min, max)
          y = stdY
          // y = map(self.randomIntFromInterval(0, 2), 0, 2, min, max)
          z = map(self.randomIntFromInterval(0, 2), 0, 2, min, max)
        }
        else if (layout === 'ring') {
          x = Math.sin( i / self.noOFCubes * Math.PI * 2 ) * ringSize
          y = stdY
          z = Math.cos( i / self.noOFCubes * Math.PI * 2 ) * ringSize
        }
        else if (layout === 'small-ring') {
          x = Math.sin( i / self.noOFCubes * Math.PI * 2 ) * ringSize / 4
          y = stdY
          z = Math.cos( i / self.noOFCubes * Math.PI * 2 ) * ringSize / 4
        }
        else if (layout === 'oort') {
          var l = self.bodies.length
          var phi = Math.acos( -1 + ( 2 * i ) / l );
					var theta = Math.sqrt( l * Math.PI ) * phi;
          x = 10 * Math.cos( theta ) * Math.sin( phi );
					y = 10 * Math.sin( theta ) * Math.sin( phi );
					z = 10 * Math.cos( phi );
        }
        TweenMax.to(body.position, self.easeTime, {
          x: x,
          y: y,
          z: z,
          ease: self.easingType,
          // ease: Elastic.easeOut.config(1, 0.3)
        });
      });
    },
    assignEffects() {
      var self = this
      self.composer = new EffectComposer(self.renderer)
      self.composer.addPass(new RenderPass(self.scene, self.camera))

      // var obj = self.getKeyByValue('FXsSliders', self.settings)

      // alert(obj.sliders[2].value)

      var params = {
				exposure: 1,
				bloomStrength: 1,
				// bloomStrength: parseFloat(obj.sliders[2].value),
				bloomThreshold: 0,
				bloomRadius: 0,
				scene: "Scene with Glow"
			}

      // New Bloom Pass - start
      self.bloomPass = new UnrealBloomPass( new THREE.Vector2( window.innerWidth, window.innerHeight ), 1.5, 0.4, 0.85 );
			self.bloomPass.threshold = params.bloomThreshold
			self.bloomPass.strength = params.bloomStrength
      self.bloomPass.radius = params.bloomRadius
      
      self.finalPass = new ShaderPass(
				new THREE.ShaderMaterial( {
					uniforms: {
						baseTexture: { value: null },
						bloomTexture: { value: self.composer.renderTarget2.texture }
					},
					vertexShader: vertexShader,
					fragmentShader: fragmentShader,
					defines: {}
				} ), 'baseTexture'
			)
			self.finalPass.needsSwap = true
      // New Bloom Pass - end

      // Org Bloom Pass - start
      // self.bloomPass = new BloomPass(
      //   1,    // strength
      //     25,   // kernel size
      //     2,    // sigma ?
      //     256,  // blur render target resolution
      // );
      // // self.bloomPass.convolutionUniforms[ "uImageIncrement" ].value.x = -0.0001
      // // self.bloomPass.convolutionUniforms[ "uImageIncrement" ].value.y = -0.005
      // // self.bloomPass.convolutionUniforms[ "uImageIncrement" ].value.x = 0
      // // self.bloomPass.convolutionUniforms[ "uImageIncrement" ].value.y = -0.002
      // Org Bloom Pass - end

      // console.log(self.bloomPass)
      // console.log(self.bloomPass.strength)

      // Anti alias
      self.fxaaPass = new ShaderPass( FXAAShader )
      var pixelRatio = self.renderer.getPixelRatio()
      self.fxaaPass.material.uniforms[ 'resolution' ].value.x = 1 / ( self.container.offsetWidth * pixelRatio );
      self.fxaaPass.material.uniforms[ 'resolution' ].value.y = 1 / ( self.container.offsetHeight * pixelRatio );
      
      // Anti alias
      self.composer.addPass( self.fxaaPass )
      
      self.composer.addPass(self.bloomPass)
      // self.composer.addPass(self.finalPass)

      self.filmPass = new FilmPass(
        0.25,   // noise intensity
        0.025,  // scanline intensity
        648,    // scanline count
        false,  // grayscale
      );
      self.filmPass.renderToScreen = true;
      self.composer.addPass(self.filmPass);

      // self.afterimagePass = new AfterimagePass();
      // self.composer.addPass( self.afterimagePass )
      
      // Pixelation
      self.pixelPass = new ShaderPass( PixelShader )
      self.pixelPass.uniforms[ "resolution" ].value = new THREE.Vector2( window.innerWidth, window.innerHeight )
      self.pixelPass.uniforms[ "resolution" ].value.multiplyScalar( window.devicePixelRatio )
      self.composer.addPass( self.pixelPass )
    },
    startMusic(id) {
      var self = this
      console.log(id)
      // console.log(sounds[id].isPlaying)
      // self.isPlaying = !self.isPlaying
      sounds.forEach((sound, index) => {
        if (typeof id === 'number') {
          if (index === id) {
            sound.audio.play()
            sound.isPlaying = !sound.isPlaying
            self.isPlaying = !self.isPlaying
          }
        }
        else {
          if (sound.isPlaying) {
            sound.audio.stop()
            sound.isPlaying = false
            self.isPlaying = false
          }
          else if (!sound.isPlaying) {
            sound.audio.play()
            sound.isPlaying = true
            self.isPlaying = true
          }
        }
      });
    },
    stopMusic(id) {
      var self = this
      sounds.forEach((sound, index) => {
        if (index === id) {
          sound.audio.stop()
          sound.isPlaying = !sound.isPlaying
        }
      });
    },
    setClickMarker(x, y, z) {
      var self = this
      if (!self.clickMarker) {
        self.shape = new THREE.SphereGeometry(0.2, 8, 8);
        self.clickMarker = new THREE.Mesh(self.shape, self.markerMaterial);
        self.scene.add(self.clickMarker);
      }
      self.clickMarker.visible = true;
      self.clickMarker.position.set(x, y, z);
    },
    clickedCommandInChild (obj) {
      var self = this
      console.log('obj: ', obj.name)
      // if (obj.name === 'camerapos') {
      //   self.toggleCamera()
      //   self.guiControls[0].enabled = !self.guiControls[0].enabled
      // }
      if (obj.name === 'ring') {
        self.distributeBodies('ring')
      }
      if (obj.name === 'ring-sm') {
        self.distributeBodies('small-ring')
      }
      if (obj.name === 'random') {
        self.distributeBodies('random')
      }
      if (obj.name === 'anim-rotate') {
        self.enableRotation = !self.enableRotation
      }
      if (obj.name === 'sun') {
        self.sun.visible = !self.sun.visible
      }
      if (obj.name === 'show-info') {
        self.showAnnotation = !self.showAnnotation
      }
      if (obj.name === 'floor') {
        self.floor.visible = !self.floor.visible
      }
      // Scale
      if (obj.name === 'scale-actual') {
        sounds.forEach((snd, index) => {
          var s = snd.r[1]
          // self.tweenObject(snd.shape.scale, 2, {x: s, y: s, z: s})
          self.tweenObject(snd.shape.position, 2, {x: index * 10, y: 5, z: 0})
        });
      }
      if (obj.name === 'front' || obj.name === 'top' || obj.name === 'right' || obj.name === 'angle' || obj.name === 'center') {
        self.changeCameraPos(obj.name)
      }
    },
    tweenOver() {
      console.log('tween is over')
    },
    // object, timing in secs, x, y, z
    tweenObject(obj, time, pos) {
      var self = this
      TweenMax.to(obj, time, {
        ease: Expo.easeOut,
        x: pos.x,
        y: pos.y,
        z: pos.z,
        onComplete: self.tweenOver()
      })
    },
    changeCameraPos(angle) {
      var self = this
      switch (angle) {
        case 'front':
          self.tweenObject(self.camera.position, 1, {x: 0, y: 0, z: stdCamDistance / 2})
          // self.camera.position.set( 0, 0, 30 )
          console.log('front')
          break
        case 'top':
          self.tweenObject(self.camera.position, 1, {x: 0, y: stdCamDistance, z: 0})
          // self.camera.position.set( 0, 30, 0 )
          break
        case 'center':
          self.tweenObject(self.camera.position, 1, {x: 0, y: 2, z: 0})
          // self.controls.update()
          break
        case 'angle':
          self.camera.position.set( -30, 10, 30 )
          break
        case 'right':
          self.camera.position.set( 30, 0, 0 )
          break
        case 'reset':
          self.resetCamera();
          break
      }
      self.camera.lookAt( 0, 0, 0 ) // TODO: look at "active" object
      self.controls.update()
      // self.render()
      // if (angle != 'reset') {
      // }
      // self.cameraPersp.position.set( 1000, 500, 1000 )
      // self.cameraOrtho.position.set( 1000, 500, 1000 )
    },
    removeClickMarker() {
      var self = this
      if (self.clickMarker) {
        self.clickMarker.visible = false;
      }
    },
    eventHappens(e) {
      var self = this

      // console.log('type of event')
      // console.log(e.type)

      // console.log(self.listener.context.state)
      // self.listener.context.resume()

      var mouseCoords = self.checkIfTouch(e)
      if (self.gplane && self.mouseConstraint) {
        var pos = self.projectOntoPlane(mouseCoords.x, mouseCoords.y, self.gplane, self.camera);
        if (pos) {
          var yDiff = self.mouseDownPos.y - pos.y
          self.setClickMarker(pos.x - yDiff**2, pos.y, pos.z, self.scene);
          self.moveJointToPoint(pos.x - yDiff**2, pos.y, pos.z);
        }
      }
      // https://stackoverflow.com/questions/38314521/change-color-of-mesh-using-mouseover-in-three-js
      // Get the picking ray from the point. https://jsfiddle.net/wilt/52ejur45/
      var mouseCoords = self.checkIfTouch(e)
      var raycaster = self.getRayCasterFromScreenCoord(mouseCoords.x, mouseCoords.y, self.camera);
      // Find the closest intersecting object
      // Now, cast the ray all render objects in the scene to see if they collide. Take the closest one.
      var intersects = raycaster.intersectObjects(self.meshes);
      
      // Intersected object
      self.intS = self.INTERSECTED
      // self.intersectedObject = self.INTERSECTED // Because intS follows specific hover rules

      // This is where an intersection is detected
      if ( intersects.length > 0 ) {
        if ( self.intS != intersects[ 0 ].object ) {
          if ( self.intS ) {            
            self.intS.material.emissive.setHex( self.intS.currentHex );
          }
          self.intS = intersects[ 0 ].object;
          self.intS.currentHex = self.intS.material.emissive.getHex();
          
          // self.intS.material.emissive.setHex( 0xffffff ); // Hover / highlight material
          // self.intS.material.emissiveIntensity = 0.05;
          
          // Store the intersected id
          self.currentId = self.intS.userData.id

          if (self.showAnnotation) {
            self.annotation.classList.add('visible')
          }
          // console.log('self.intS: ', self.intS.userData.id)
          
          // If type of event is mousemove do not play sound. Only on mousedown
          console.log(e.type)
          if (e.type === 'mousedown' || e.type === 'touchstart') {
            console.log(e.type)
            if (sounds[self.currentId].isPlaying) {
              self.stopMusic(self.currentId)
            }
            else {
              self.startMusic(self.currentId)
              // This is is where a planet is clicked
              var pos = self.meshes[self.currentId].position;
              self.tweenObject(self.controls.target, 1, {x: pos.x, y: pos.y, z: pos.z})
            }
          }
        }

        self.intersectedObject = self.intS

        // Change cursor
        document.body.style.cursor = 'pointer'
      }
      else {
        if ( self.intS ) {
          self.intS.material.emissive.setHex( self.intS.currentHex );
        }
        self.intS = null;
        // self.intersectedObject = null;

        if (self.showAnnotation) {
          self.annotation.classList.remove('visible')
        }
        
        // Change cursor
        document.body.style.cursor = 'default'
      }
      // self.highlightShape(closest)
      self.meshes.forEach(element => {
        // console.log(element.material)
        if (element != self.intS) {
          // element.material.emissive.setHex( 0x000000 );
          // self.intS.material.emissiveIntensity = 0.025;
        }
        // console.log(element.currentHex)
      });

      // Set style for intS annotation
      // if (self.intersectedObject === null) {
      //   self.annotation.classList.remove('visible')
      // }
      // else {
      //   self.annotation.classList.add('visible')
      // }
    },
    turnOffAgain () {
      var self = this
    },
    onMouseDown(e) {
      var self = this
      // Check if touch or not
      var mouseCoords = self.checkIfTouch(e)
      // console.log('mouse coords: ', mouseCoords)
      // var entity = self.findNearestIntersectingObject(e.clientX, e.clientY, self.camera, self.meshes);
      var entity = self.findNearestIntersectingObject(mouseCoords.x, mouseCoords.y, self.camera, self.meshes);
      var pos = entity.point;
      self.mouseDownPos = pos
      if (pos && entity.object.geometry instanceof THREE.BoxGeometry) {
        self.constraintDown = true;
        // Set marker on contact point
        // alert('yes')
        self.setClickMarker(pos.x, pos.y, pos.z, self.scene); 

        // Set the movement plane
        self.setScreenPerpCenter(pos, self.camera);

        var idx = self.meshes.indexOf(entity.object);
        if (idx !== -1) {
          self.addMouseConstraint(pos.x, pos.y, pos.z, self.bodies[idx]);
        }
      }
    },
    // This function creates a virtual movement plane for the mouseJoint to move in
    setScreenPerpCenter(point, camera) {
      var self = this
      // If it does not exist, create a new one
      if (!self.gplane) {
        var planeGeo = new THREE.PlaneGeometry(100, 100);
        var hide = new THREE.MeshLambertMaterial({
          opacity: 0,
          transparent: true
        });
        self.gplane = new THREE.Mesh(planeGeo, hide);
        // plane.visible = false; // Hide it.. 
        self.scene.add(self.gplane);
      }

      // Center at mouse position
      self.gplane.position.copy(point);

      // Make it face toward the camera
      self.gplane.quaternion.copy(camera.quaternion);
    },
    onMouseUp(e) {
      var self = this
      // console.log(e)
      self.constraintDown = false
      self.mouseDownPos = null
      // remove the marker
      self.removeClickMarker()

      // Send the remove mouse joint to server
      // self.removeJointConstraint()
    },
    projectOntoPlane(screenX, screenY, thePlane, camera) {
      var self = this
      var x = screenX;
      var y = screenY;
      var now = new Date().getTime();
      // project mouse to that plane
      var hit = self.findNearestIntersectingObject(screenX, screenY, camera, [thePlane]);
      self.lastx = x;
      self.lasty = y;
      self.last = now;
      if (hit) {
        return hit.point;
      }
      return false;
    },
    highlightShape (shape) {
      var self = this
      // console.log('shape: ', shape.object)
      // store color of closest object (for later restoration)
      // shape.currentHex = INTERSECTED.material.color.getHex();
      // // set a new color for closest object
      // shape.object.material.color.setHex(0xffff00);
    },
    findNearestIntersectingObject(clientX, clientY, camera, objects) {
      var self = this
      // Get the picking ray from the point
      var raycaster = self.getRayCasterFromScreenCoord(clientX, clientY, camera);

      // Find the closest intersecting object
      // Now, cast the ray all render objects in the scene to see if they collide. Take the closest one.
      var hits = raycaster.intersectObjects(objects);
      var closest = false;
      if (hits.length > 0) {
        closest = hits[0];
        self.highlightShape(closest)
      }
      return closest;
    },
    // Function that returns a raycaster to use to find intersecting objects
    // in a scene given screen pos and a camera, and a projector
    getRayCasterFromScreenCoord (screenX, screenY, camera) {
      var self = this
      var raycaster = new THREE.Raycaster()
      var mouse3D = new THREE.Vector3();
      // Get 3D point form the client x y
      mouse3D.x = (screenX / window.innerWidth) * 2 - 1;
      mouse3D.y = -(screenY / window.innerHeight) * 2 + 1;
      mouse3D.z = 0.5;
      raycaster.setFromCamera(mouse3D, camera)
      return raycaster
    },
    updatePhysics() {
      var self = this
      self.world.step(self.dt);
      for (var i = 0; i !== self.meshes.length; i++) {
        self.meshes[i].position.copy(self.bodies[i].position);
        self.meshes[i].quaternion.copy(self.bodies[i].quaternion);
      }
    },
    checkRotation(){
      var rotSpeed = .002
      var x = this.camera.position.x,
        y = this.camera.position.y,
        z = this.camera.position.z;

      this.camera.position.x = x * Math.cos(rotSpeed) + z * Math.sin(rotSpeed);
      this.camera.position.z = z * Math.cos(rotSpeed) - x * Math.sin(rotSpeed); 

      this.camera.lookAt(new THREE.Vector3(0,0,0));
    },
    render() {
      var self = this
      const delta = self.clock.getDelta()
      self.controls.update( delta )
      
      // This is physics - update start
      // self.updatePhysics()
      // This is physics - update end

      if (self.enableRotation)
        self.checkRotation()

      // self.groupOfObjects.rotation.x++;
      // self.camera.rotation.x++;
      // if (rotateCamera) {
      //   self.camera.rotation += 0.001;
      // }

      // self.controls.update( delta );
      // console.log(`running this route: ${this.$route.path}`)
      // self.renderer.render(self.scene, self.camera);

      // Animate noise ball
      // self.nMaterial.uniforms[ 'time' ].value = self.animTime * ( Date.now() - self.start );
      // self.nMaterial.uniforms[ 'weight' ].value = 10.0 * ( 0.5 + 0.5 * Math.sin( 0.00025 * ( Date.now() - self.start ) ) );

      // Post-processing
      // self.renderer.render(self.scene, self.camera);
      self.composer.render()
      // if (self.renderComposer) {
      //   if (vrEnabled) {
      //     self.renderer.render(self.scene, self.camera);
      //   }
      //   else {
      //     self.composer.render()
      //   }

      //   // VR Experiment - start
      //   // self.renderer.setAnimationLoop( self.animate.bind(this) );
      //   // if (vrEnabled) {
      //   //   self.renderer.setAnimationLoop( self.render );
      //   // }
      //   // else {
      //   //   self.composer.render()
      //   // }
      //   // VR Experiment - end
      // }

      // Scaling
      // log sound analysis
      sounds.forEach((element, index) => {
        // element.shape.scale.y = sounds[0].obj.analyser.getFrequencyData()
        
        var freq = sounds[index].analyser.getFrequencyData()[0]
        var scaledVal = map(freq, 0, 256, 1, scaleVal)
        // element.shape.scale.set(scaledVal * 2, scaledVal * 2, scaledVal * 2)
        
        // Scaling of Physics body
        // var boxShape = self.bodies[index].shapes[0]
        // boxShape.halfExtents.set(scaledVal, scaledVal, scaledVal);
        // // boxShape.halfExtents.set(1, scaledVal, 1);
        // boxShape.updateConvexPolyhedronRepresentation();
        // Scaling of Physics body - end

        // Scaling of Mesh according to volume
        // X, Y and Z
        // TweenMax.to(element.shape.scale, self.easeTime, {
        //   x: scaledVal,
        //   y: scaledVal,
        //   z: scaledVal,
        //   ease: Sine.easeOut
        // })
        
        // One dimension
        // TweenMax.to(element.shape.scale, self.easeTime, {y: scaledVal, ease: Bounce.easeOut})
        
        // boxShape.computeBoundingSphereRadius();
        // boxShape.computeAABB();
        // boxShape.updateMassProperties();
        // console.log('body: ', self.bodies[index].shapes)
        // console.log('body: ', boxShape.shapes[0])
        // self.bodies[index].shape.halfExtents.y = scaledVal
        // self.bodies[index].shape.boundingSphereRadiusNeedsUpdate = true
        // self.bodies[index].shape.updateConvexPolyhedronRepresentation()
      });
      // console.log('sound: ', sounds[0].obj.analyser.getFrequencyData().length)
      // console.log('sound: ', sounds[0].shape) 
      
      if (self.debug) {
        self.$refs.debug_text.innerHTML = 'rendering'
        // console.log('rendering: ')
      }

      if (vrEnabled) {
        self.renderer.setAnimationLoop( self.render.bind(this) );
      }
      else {
        self.reqAnim = requestAnimationFrame( self.render.bind(this) );
      }

      // Update overlay. codepen.io/konradstudio/pen/pogVPrB
      // self.updateAnnotationOpacity();
      // console.log('draw tooltip', self.intersectedObject)

      if (self.intersectedObject && self.showAnnotation) {
        self.updateScreenPosition();
      }
    },
    updateAnnotationOpacity() {
      var self = this;
      // var mesh = self.meshes[0];
      // self.currentId = self.intS.userData.id
      const meshDistance = self.camera.position.distanceTo(mesh.position);
      const spriteDistance = self.camera.position.distanceTo(sprite.position);
      self.spriteBehindObject = spriteDistance > meshDistance;
      sprite.material.opacity = spriteBehindObject ? 0.25 : 1;

      // Do you want a number that changes size according to its position?
      // Comment out the following line and the `::before` pseudo-element.
      sprite.material.opacity = 0;
    },
    updateScreenPosition() {
      var self = this;

      // const vector = new THREE.Vector3(0, 0, 0);
      if (self.intersectedObject === null) {
        return
      }
      var mesh = self.intersectedObject;
      // var mesh = self.meshes[0];
      const vector = mesh.position.clone();
      const canvas = self.renderer.domElement;

      vector.project(self.camera);

      vector.x = Math.round((0.5 + vector.x / 2) * (canvas.width / window.devicePixelRatio));
      vector.y = Math.round((0.5 - vector.y / 2) * (canvas.height / window.devicePixelRatio));

      self.annotation.innerHTML = sounds[self.currentId].name;
      self.annotation.style.top = `${vector.y - 84}px`;
      self.annotation.style.left = `${vector.x}px`;
      // self.annotation.style.opacity = spriteBehindObject ? 0.25 : 1;
    },
    map_range(value, low1, high1, low2, high2) {
      return low2 + (high2 - low2) * (value - low1) / (high1 - low1);
    },
    createFloor() {
      var self = this 
      // floor
      var geometry = new THREE.PlaneGeometry(5000, 5000, 100, 100);
      //geometry.applyMatrix( new THREE.Matrix4().makeRotationX( -Math.PI / 2 ) );
      var material = new THREE.MeshLambertMaterial({
      // var material = new THREE.MeshPhongMaterial({
        // color: 0xffffff,
        color: 0x151515,
        // side: THREE.DoubleSide,
        // color: 0x050505
      });
      self.markerMaterial = new THREE.MeshLambertMaterial({
        color: 0xff0000
      });
      
      //THREE.ColorUtils.adjustHSV( material.color, 0, 0, 0.9 );
      self.floor = new THREE.Mesh(geometry, material);
      
      // self.floor.quaternion.setFromAxisAngle(new THREE.Vector3(1, 0, 0), -Math.PI / 2);
      // self.floor.castShadow = true;
      self.floor.receiveShadow = true;
      self.floor.rotation.x = -Math.PI / 2.0;
      
      self.floor.position.y = -1;
      self.floor.visible = false;
      self.scene.add(self.floor);

      //Create a plane that receives shadows (but does not cast them)
      // var planeGeometry = new THREE.PlaneBufferGeometry( 200, 200, 32, 32 );
      // var planeMaterial = new THREE.MeshStandardMaterial( { color: 0x00ff00 } )
      // var plane = new THREE.Mesh( planeGeometry, planeMaterial );
      // plane.receiveShadow = true;
      // self.scene.add( plane );
    },
    // This is where all cubes are being created
    createBodies () {
      var self = this 
      var mass = 5
      // var shape = new CANNON.Box(new CANNON.Vec3(bS, bS, bS))
      self.noOFCubes = sounds.length
      for (var i = 0; i < self.noOFCubes; i++) {
        var shape;
        if (sounds[i].type === 'sphere') {
          shape = new CANNON.Sphere(bS)
        }
        else {
          shape = new CANNON.Box(new CANNON.Vec3(bS, bS, bS))
        }

        // Create texture
        var texture = new THREE.TextureLoader().load( sounds[i].texture )

        self.materialObject = new THREE.MeshStandardMaterial({
          color: 0x333333,
          roughness: 0,
          // metalness: 0,
          emissive: sounds[i].type === 'sun' ? '#F8CE3B' : '#000000',
          // emissive: '#F8CE3B',
          // emissiveIntensity: 0.025,
          map: texture ? texture : '',
          // flatShading: true
        })

        var rX, rY, rZ
        if (sounds[i].type === 'sun') {
          rX = 0
          rY = 0
          rZ = 0
        }
        else {
          // Distribute in circle
          rX = Math.sin( i / (self.noOFCubes - 1) * Math.PI * 2 ) * ringSize
          rY = 0
          rZ = Math.cos( i / (self.noOFCubes - 1) * Math.PI * 2 ) * ringSize
        }


        // This is physics - init start
        //
        // // First Cannon
        // var boxBody = new CANNON.Body({
        //   mass: mass
        // })
        // boxBody.addShape(shape)
        // boxBody.position.set(rX, rY, rZ)
        // self.world.addBody(boxBody)
        // self.bodies.push(boxBody)
        //
        // This is physics - init end

        // On collision
        // boxBody.addEventListener('collide', () => {
        //   console.log('colliding')
        // })

        // Geometry
        // for some reason geometry is double of Cannon body
        var planetGeom = null;
        // Create sphere
        planetGeom = new THREE.SphereBufferGeometry( sounds[i].r[0], reso, reso )
 
        var planetMesh = new THREE.Mesh(planetGeom, self.materialObject);
        planetMesh.position.set(rX, rY, rZ)
        planetMesh.castShadow = true;
        // planetMesh.receiveShadow = true;
        self.meshes.push(planetMesh)
        
        sounds[i].shape = planetMesh
        self.scene.add(planetMesh)

        if (sounds[i].type === 'sun') {
          self.sun = planetMesh
          // planetMesh.visible = false
        }

        // Assign ID to mesh
        planetMesh.userData.id = i

        // Setup sound
        // var snd = self.soundObjs[i]
        // if (i != 1) {
        //   return
        // }
        self.audioLoader[i] = new THREE.AudioLoader()

        var sound = new THREE.PositionalAudio( self.listener );
        sounds[i].audio = sound
        var sndPath = path + sounds[i].filename
        
        // Load sound
        self.loadSound(sound, i, sndPath)
        
        planetMesh.add( sound );
        
        // Load text
        if (shouldShowText) {
          self.loadText(i, sounds[i].name, {x: rX, y: rY, z: rZ})
        }
        
        // console.log('it is 2, ', sndPath)

        // if (i === 0) {
        //   console.log('it is 2, ', path + sounds[i])
        //   var snd = new THREE.PositionalAudio( self.listener );
        //   self.audioLoader[i].load( path + sounds[0], function ( buffer ) {
        //     snd.setBuffer( buffer );
        //     snd.setLoop( true );
        //     snd.setRefDistance( 20 );
        //     snd.setVolume( 0.5 );
        //     snd.play();
        //   } );
        //   // console.log(boxBody)
        //   // boxBody.add( snd )
        //   cubeMesh.add( snd )
        // } else {
        //   console.log('it is 2, ', path + sounds[i])
        //   var sound2 = new THREE.PositionalAudio( self.listener );
        //   self.audioLoader[i].load( path + sounds[1], function ( buffer ) {
        //     sound2.setBuffer( buffer );
        //     sound2.setRefDistance( 20 );
        //     sound2.play();

        //   } );
        //   cubeMesh.add( sound2 );
        // }
      }
    },
    loadText(i, name, pos) {
      var self = this
      var textGeo
      self.textMaterial = new THREE.MeshPhongMaterial({ 
        color: 0x0000ff,
        flatShading: true,
        emissive: 0x0000ff
      })
      // var fontInstance
      var loadFont = function () {
        textGeo = new THREE.TextGeometry( name, {
          font: self.fonts[ self.inc ],
          size: 4,
          height: 1,
          curveSegments: 0,
          bevelThickness: 0,
          bevelSize: 0, // 1.5
          bevelEnabled: true
        })
        textGeo.computeBoundingBox()
        textGeo.computeVertexNormals()
        textGeo = new THREE.BufferGeometry().fromGeometry( textGeo )
        console.log('textGeo: ', textGeo)
        console.log('pos x: ' + pos.x + ', pos y: ' + pos.y + ', pos z: ' + pos.z)

        var textMesh = new THREE.Mesh( textGeo, self.textMaterial )
        var centerOffset = - 0.5 * ( textGeo.boundingBox.max.x - textGeo.boundingBox.min.x )
        // textMesh.position.set(new THREE.Vector3(pos.x, 2, pos.z))
        textMesh.position.x = pos.x + centerOffset
        // textMesh.position.x = pos.x
        textMesh.position.y = 10
        textMesh.position.z = pos.z
        // textMesh.position.x = 0
        textMesh.lookAt(self.camera.position)
        sounds[i].text = textMesh
        self.scene.add(textMesh)
        // Iterate font inc load
        self.inc++
      }
      // var fontPath = 'fonts/avenir_ff/Avenir LT Std 65 Medium_Bold.json'
      var fontPath = 'fonts/helvetiker.json'
      self.fontLoader.load( fontPath, function ( response ) {
        self.fonts.push( response )
        // console.log('response: ', response)
        loadFont()
      })
    },
    loadSound(sound, i, sndPath) {
      var self = this
      self.audioLoader[i].load( sndPath, function ( buffer ) {
        sound.setBuffer( buffer )
        sound.setRefDistance( 20 )
        sound.setLoop( true )
        // sound.play()
        sounds[i].audio = sound
        // console.log('its working alright')
      })
      sounds[i].analyser = new THREE.AudioAnalyser( sound, 32 );
      // console.log()
    },
    addMouseConstraint (x, y, z, body) {
      var self = this
      // The cannon body constrained by the mouse joint
      self.constrainedBody = body;

      // Vector to the clicked point, relative to the body
      var v1 = new CANNON.Vec3(x, y, z).vsub(self.constrainedBody.position);

      // Apply anti-quaternion to vector to tranform it into the local body coordinate system
      var antiRot = self.constrainedBody.quaternion.inverse();
      var pivot = antiRot.vmult(v1); // pivot is not in local body coordinates

      // Move the cannon click marker particle to the click position
      self.jointBody.position.set(x, y, z);

      // Create a new constraint
      // The pivot for the jointBody is zero
      self.mouseConstraint = new CANNON.PointToPointConstraint(self.constrainedBody, pivot, self.jointBody, new CANNON.Vec3(0, 0, 0));

      // Add the constriant to world
      self.world.addConstraint(self.mouseConstraint);
    },
    moveJointToPoint (x, y, z) {
      var self = this
      // Move the joint body to a new position
      self.jointBody.position.set(x, y, z)
      self.mouseConstraint.update()
    },
    removeJointConstraint () {
      var self = this
      // Remove constriant from world
      self.world.removeConstraint(self.mouseConstraint)
      self.mouseConstraint = false
    },
    onWindowResize() {
      var self = this
      self.camera.aspect = ( window.innerWidth / window.innerHeight )
      self.camera.updateProjectionMatrix()
      self.renderer.setSize( window.innerWidth, window.innerHeight )
    },
    setOrbitControls () {
      var self = this
      if (self.isWalking) {
        // FPS Controls
        self.controls = new FirstPersonControls( self.camera, self.renderer.domElement );
        self.controls.movementSpeed = 20;
        self.controls.lookSpeed = 0.05;
        self.controls.noFly = true;
        self.controls.lookVertical = false;
        // FPS Controls - end
      }
      else {
        // Orbit controls
        self.controls = new OrbitControls( self.camera, self.renderer.domElement );
        self.controls.enablePan = false
        // self.controls.enableRotate = false
        self.controls.enableDamping = true
        self.controls.zoomSpeed = 0.2
        self.controls.addEventListener('change', () => {
          // console.log('x: ', self.camera.position.x)
          // console.log('x: ', self.camera.position.y)
          // console.log('z: ', self.camera.position.z)
          if (shouldShowText) {
            sounds.forEach(element => {
              element.text.lookAt(self.camera.position)
            });
          }
        })

        // Change orbit point
        // self.controls.target = self.meshes[1].position;
        // Orbit controls - end
      }
    },
    listenForKeyEvents() {
      var self = this
      window.addEventListener( 'keydown', function ( event ) {
        console.log('key code: ', event.keyCode)
        var key
        var target = event.target || event.srcElement
        var isSpecialKey
        if (window.event) {
          key = window.event.keyCode
          // isSpecialKey = !!window.event.shiftKey // typecast to boolean
          // isSpecialKey = !!window.event.metaKey // Cmd
          isSpecialKey = !!window.event.altKey // typecast to boolean
        } else {
          key = ev.which
          // isSpecialKey = !!ev.shiftKey
          // isSpecialKey = !!ev.metaKey
          isSpecialKey = !!ev.altKey
        }
        
        // console.log('where it stems from: ', event)
        if (target.tagName === 'INPUT' && key !== 27) {
          return
        }

        // Key is pressed
        if (isSpecialKey) {
          event.preventDefault()
          switch ( key ) {
            case 77: // M
              self.quickSearch['enabled'] = !self.quickSearch['enabled']
              break
            case 67: // C
              self.toggleCamera()
              break
            case 49: // 1
              self.changeCameraPos('front')
              break
            case 50: // 2
              self.changeCameraPos('top')
              break
            case 51: // 3
              self.changeCameraPos('angle')
              break
            case 52: // 4
              self.distributeBodies('random')
              break
            case 53: // 5
              self.distributeBodies('ring')
              break
            case 54: // 6
              self.distributeBodies('small-ring')
              break
            case 55: // 7
              self.distributeBodies('oort')
              break
            case 82: // R
              self.changeCameraPos('reset')
              break
            case 83: // S – take a screenshot
              self.takeScreenshot()
              break
            case 32: // S – take a screenshot
              // self.doubleTapProvoked(event)
              break
          }
        }
        // Global keys
        switch ( key ) {
          case 27: // Escape key
            self.quickSearch['enabled'] = false
            break
        }
        // if ( event.keyCode == 49 && event.keyCode == 16 ) {
        //   self.toggleCamera()
        // }
        // else if () {

        // }
      })
    },
    initCannon () {
      var self = this
      // Setup our world
      self.world = new CANNON.World();
      self.world.quatNormalizeSkip = 0;
      self.world.quatNormalizeFast = false;

      self.world.gravity.set(0, -10, 0);
      // self.world.gravity.set(0, 0, 0); // No gravity
      self.world.broadphase = new CANNON.NaiveBroadphase();

      // Create a plane
      var groundShape = new CANNON.Plane();
      var groundBody = new CANNON.Body({
        mass: 0
      });
      groundBody.addShape(groundShape);
      groundBody.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), -Math.PI / 2);
      self.world.addBody(groundBody);

      // Joint body
      var shape = new CANNON.Sphere(0.1);
      self.jointBody = new CANNON.Body({
        mass: 0
      });
      self.jointBody.addShape(shape);
      self.jointBody.collisionFilterGroup = 0;
      self.jointBody.collisionFilterMask = 0;
      self.world.addBody(self.jointBody)
    },
    addGuides () {
      var self = this
      
      // Axes
      var axesHelper = new THREE.AxesHelper( 5 );
      this.scene.add( axesHelper );
      console.log('adding guides')

      // Grid
      // var helper = new THREE.GridHelper( 1000, 100, 0x444444, 0x444444 )
      // helper.position.y = 0.1
      // self.scene.add( helper )
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>

$breakp-xs: 480px;
$breakp-sm: 576px;
$breakp-md: 768px;
$breakp-lg: 992px;
$breakp-xl: 1200px;
$breakp-xxl: 1600px;

$annotate-w-lg: 200px;
$color-annotate: rgba(10, 10, 10, 0.8);

// Annotation
.annotation {
  width: $annotate-w-lg;
  position: absolute;
  top: 0;
  margin-left: calc((#{$annotate-w-lg} / 2) * -1);
  z-index: 1;
  // margin-left: 15px;
  // margin-top: 15px;
  padding: 1em;
  color: #fff;
  background: $color-annotate;
  // background: rgba(0, 0, 0, 0.8);
  border-radius: 2px;
  font-size: 12px;
  line-height: 1.2;
  transition: opacity, background .5s;
  box-sizing: border-box;
  text-align: center;
  // opacity: 0;
  &.visible {
    // opacity: 1;
    background: lighten($color-annotate, 4);
  }
  // &::before {
  //   content: '1';
  //   position: absolute;
  //   top: -30px;
  //   left: -30px;
  //   width: 30px;
  //   height: 30px;
  //   border: 2px solid #fff;
  //   border-radius: 50%;
  //   font-size: 16px;
  //   line-height: 30px;
  //   text-align: center;
  //   background: rgba(0, 0, 0, 0.8);
  // }
}

#number {
  position: absolute;
  z-index: -1;
}

</style>
