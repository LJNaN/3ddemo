<template>
  <div id="canvasContent"></div>
  <div class="lightBtn">
    <button v-for="item in lights" @click.stop="handleLight(item)">{{ item.name }} {{ item.open ? '已开' : '已关'
    }}</button>
  </div>
  <div class="lightBtn" style="margin-top: 30px">
    <button @click.stop="handleCamera">正交摄像机(与平行光阴影范围同步) {{ isOrthographicCamera ? '已开' : '已关' }}</button>
  </div>
  <div class="lightBtn" style="margin-top: 60px">
    <button @click.stop="handleFloor">镜面反射 {{ isMirrorFloor ? '已开' : '已关' }}</button>
  </div>
  <div class="lightBtn" style="margin-top: 90px">
    <button @click.stop="handleFog">雾 {{ isFog ? '已开' : '已关' }}</button>
    <button @click.stop="handleSnow">雪(粒子) {{ isSnow ? '已开' : '已关' }}</button>
  </div>
  <div class="lightBtn" style="margin-top: 120px">
    <button @click.stop="modelActive(0)">人物 停</button>
    <button @click.stop="modelActive(3)">人物 走</button>
    <button @click.stop="modelActive(1)">人物 跑</button>
  </div>
  <div class="lightBtn" style="margin-top: 150px">
    <button @click.stop="handleEffect">后处理 {{ isAfterEffect ? '已开' : '已关' }}</button>
  </div>
</template>

<script setup>
import { onBeforeMount, onMounted, watch, ref, reactive } from 'vue';
import * as THREE from 'three';
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { RectAreaLightUniformsLib } from 'three/examples/jsm/lights/RectAreaLightUniformsLib';
import { Reflector } from "three/examples/jsm/objects/Reflector";
import { Water as ThreeWater } from "three/examples/jsm/objects/Water";
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { CSS2DObject, CSS2DRenderer } from 'three/examples/jsm/renderers/CSS2DRenderer';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { AfterimagePass } from 'three/examples/jsm/postprocessing/AfterimagePass.js';
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min'

const lights = ref([
  { name: '环境光', open: true },
  { name: '平行光', open: true },
  { name: '点光源', open: true },
  { name: '聚光灯', open: true },
  { name: '平面光', open: true }
])

let windowHeight = document.body.clientHeight
let windowWidth = document.body.clientWidth

let innerObject = {}


let
  clock = null,
  myStats = null,
  Camera = null,
  Scene = null,
  Controls = null,
  Renderer = null,
  LabelRenderer = null,
  AmbientLight = null,
  DirectionalLight = null,
  PointLight = null,
  SpotLight = null,
  RectAreaLight = null,
  isOrthographicCamera = ref(false),
  isMirrorFloor = ref(false),
  isFog = ref(false),
  isSnow = ref(true),
  composer = null,
  afterimagePass = null,
  isAfterEffect = ref(false),

  animateFlag = (() => { }),
  beforeChoose = (() => { }),
  afterChoose = (() => { }),
  clickMeshArr = [],
  clickTempMaterial = null



onMounted(() => {
  initRenderer()
  initCamera()
  initScene()
  initControls()
  initLights()
  initFog()
  demo1()

  animate()

  // Controls.addEventListener('end', () => {
  //   
  // })
})


function initRenderer() {
  Renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  });
  Renderer.shadowMap.enabled = true
  Renderer.shadowMap.type = THREE.PCFSoftShadowMap
  Renderer.gammaOutput = true
  Renderer.gammaFactor = 2.2
  Renderer.toneMapping = THREE.LinearToneMapping
  Renderer.toneMappingExposure = 1
  Renderer.setSize(windowWidth, windowHeight);
  document.getElementById('canvasContent').appendChild(Renderer.domElement);


}

function initCamera() {
  const cameraOption = {
    fov: 45,
    aspect: windowWidth / windowHeight,
    near: 0.1,
    far: 50000
  }

  const OrthographicCameraOption = {
    near: 0.1,
    far: 5000,
    top: 40,
    bottom: -40,
    right: 40,
    left: -40
  }

  if (Camera) {
    const temp = Camera.clone()

    if (isOrthographicCamera.value) {
      Camera = new THREE.OrthographicCamera(
        OrthographicCameraOption.left,
        OrthographicCameraOption.right,
        OrthographicCameraOption.top,
        OrthographicCameraOption.bottom,
        OrthographicCameraOption.near,
        OrthographicCameraOption.far
      )

      Camera.position.set(9, 24, 13)
      Camera.rotation.set(temp.rotation)
      Camera.scale.set(temp.scale.x, temp.scale.y, temp.scale.z)
      Camera.zoom = temp.zoom
    } else {
      Camera = new THREE.PerspectiveCamera(cameraOption.fov, cameraOption.aspect, cameraOption.near, cameraOption.far);

      Camera.position.set(temp.position.x, temp.position.y, temp.position.z)
      Camera.rotation.set(temp.rotation)
      Camera.scale.set(temp.scale.x, temp.scale.y, temp.scale.z)
      Camera.zoom = temp.zoom
    }
  } else {
    if (isOrthographicCamera.value) {
      Camera = new THREE.OrthographicCamera(
        OrthographicCameraOption.left,
        OrthographicCameraOption.right,
        OrthographicCameraOption.top,
        OrthographicCameraOption.bottom,
        OrthographicCameraOption.near,
        OrthographicCameraOption.far
      )
      Camera.position.set(9, 24, 13)
    } else {
      Camera = new THREE.PerspectiveCamera(cameraOption.fov, cameraOption.aspect, cameraOption.near, cameraOption.far);
    }
  }
  initControls()
}

function initScene() {
  Scene = new THREE.Scene();
  RectAreaLightUniformsLib.init();
  const skyArr = [
    '/texture/px.png',
    '/texture/nx.png',
    '/texture/py.png',
    '/texture/ny.png',
    '/texture/pz.png',
    '/texture/nz.png'
  ]

  Scene.background = new THREE.CubeTextureLoader().load(skyArr)
  
  clock = new THREE.Clock()

  myStats = new Stats();
  document.getElementById('canvasContent').appendChild( myStats.dom );

}

function initControls() {
  Controls = new OrbitControls(Camera, Renderer.domElement)
  // Controls.autoRotate = true
  // Controls.autoRotateSpeed = 0.5
}

function initLights() {
  AmbientLight = new THREE.AmbientLight(0x404040, 3);
  Scene.add(AmbientLight);

  DirectionalLight = new THREE.DirectionalLight(0xffffff, 0.2);
  DirectionalLight.position.set(9, 24, 13)
  DirectionalLight.castShadow = true
  DirectionalLight.shadow.mapSize.width = 2048;
  DirectionalLight.shadow.mapSize.height = 2048;
  DirectionalLight.shadow.camera.near = 0.1;
  DirectionalLight.shadow.camera.far = 5000;
  DirectionalLight.shadow.camera.top = 40;
  DirectionalLight.shadow.camera.bottom = -40;
  DirectionalLight.shadow.camera.right = 40;
  DirectionalLight.shadow.camera.left = -40;
  Scene.add(DirectionalLight);

  PointLight = new THREE.PointLight(0x660000, 1, 100);
  PointLight.position.set(-10, 20, 10);
  PointLight.shadow.mapSize.width = 2048;
  PointLight.shadow.mapSize.height = 2048;
  PointLight.castShadow = true
  Scene.add(PointLight)

  SpotLight = new THREE.SpotLight(0x00ff00, 0.2);
  SpotLight.position.set(10, 5, -10);
  SpotLight.castShadow = true;
  SpotLight.angle = Math.PI / 4
  SpotLight.shadow.mapSize.width = 1024;
  SpotLight.shadow.mapSize.height = 1024;
  SpotLight.shadow.camera.near = 0.1;
  SpotLight.shadow.camera.far = 5000;
  SpotLight.shadow.camera.fov = 45;
  Scene.add(SpotLight);


  RectAreaLight = new THREE.RectAreaLight(0xffffff, 4, 6, 10);
  RectAreaLight.position.set(0, 5, -31);
  RectAreaLight.rotateX(Math.PI)
  Scene.add(RectAreaLight);

  // Scene.add(new RectAreaLightHelper(RectAreaLight))
}

function initFloor() {
  let plane = null

  if(innerObject.plane) {
    const floor = Scene.children.find(e => e.name === 'floor')
    Scene.remove(floor)
  }

  const planeGeometry = new THREE.PlaneGeometry(100, 100)

  if (!isMirrorFloor.value) {
    const planeMaterial = new THREE.MeshLambertMaterial({ color: 0xdddddd })
    plane = new THREE.Mesh(planeGeometry, planeMaterial)
  } else {
    plane = new Reflector(
      planeGeometry,
      {
        clipBias: 0.03,
        textureWidth: window.innerWidth * window.devicePixelRatio,
        textureHeight: window.innerHeight * window.devicePixelRatio,
        color: new THREE.Color(0x888888),
      }
    )
  }

  plane.name = 'floor'
  plane.receiveShadow = true
  plane.rotateX(-Math.PI / 2)
  innerObject.plane = plane
  Scene.add(innerObject.plane)
}

function demo1() {
  innerObject = {}
  innerObject.interactive = []
  animateFlag = null
  Camera.position.set(20, 8, 20)

  initFloor()
  

  // 点击变色的三个box
  const geometry1 = new THREE.BoxGeometry(1, 1, 1)
  const material1 = new THREE.MeshLambertMaterial({ color: 0xb28501 })
  innerObject.box1 = new THREE.Mesh(geometry1, material1)
  innerObject.box1.castShadow = true
  innerObject.box1.position.set(2, 1, 5)
  Scene.add(innerObject.box1)

  const geometry2 = new THREE.BoxGeometry(2, 2, 2)
  const material2 = new THREE.MeshLambertMaterial({ color: 0x8679ff })
  innerObject.box2 = new THREE.Mesh(geometry2, material2)
  innerObject.box2.castShadow = true
  innerObject.box2.position.set(3, 3, 3)
  Scene.add(innerObject.box2)

  const geometry3 = new THREE.BoxGeometry(1.5, 2, 1.5)
  const material3 = new THREE.MeshLambertMaterial({ color: 0x9b1c27 })
  innerObject.box3 = new THREE.Mesh(geometry3, material3)
  innerObject.box3.castShadow = true
  innerObject.box3.position.set(8, 2, 3)
  Scene.add(innerObject.box3)

  // 平面光 地板和box
  const geometry4 = new THREE.BoxGeometry(20, 0.1, 20);
  const material4 = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.3, metalness: 0 });
  const floor = new THREE.Mesh(geometry4, material4);
  floor.position.set(0, 0.2, -30)
  innerObject.floor = floor
  Scene.add(floor);

  const geometry5 = new THREE.BoxGeometry(2, 2, 2);
  const material5 = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0, metalness: 0 });
  const box5 = new THREE.Mesh(geometry5, material5);
  box5.position.set(0, 1, -30);
  innerObject.box5 = box5
  Scene.add(box5);

  // 6个面6个纹理
  const geometry6 = new THREE.BoxGeometry(3, 3, 3)
  const box6ImgSrcArr = [
    "/texture/1.png",
    "/texture/2.png",
    "/texture/3.png",
    "/texture/4.png",
    "/texture/5.png",
    "/texture/6.png"
  ]
  let material6Arr = []
  box6ImgSrcArr.forEach((src) => {
    material6Arr.push(new THREE.MeshLambertMaterial({
      map: new THREE.TextureLoader().load(src)
    }))
  })
  const box6 = new THREE.Mesh(geometry6, material6Arr)
  box6.castShadow = true
  box6.position.set(-10, 3, 0)
  innerObject.box6 = box6
  Scene.add(box6)


  // 4个不同的材质
  const geometry7 = new THREE.BoxGeometry(2, 2, 2);
  const material7 = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  const box7 = new THREE.Mesh(geometry7, material7);
  box7.position.set(-15, 1, 10);
  box7.castShadow = true
  innerObject.box7 = box7
  Scene.add(box7);

  const material8 = new THREE.MeshPhongMaterial({ color: 0xff0000 });
  const box8 = new THREE.Mesh(geometry7, material8);
  box8.position.set(-15, 1, 14);
  box8.castShadow = true
  innerObject.box8 = box8
  Scene.add(box8);

  const material9 = new THREE.MeshLambertMaterial({ color: 0xff0000 });
  const box9 = new THREE.Mesh(geometry7, material9);
  box9.position.set(-15, 1, 18);
  box9.castShadow = true
  innerObject.box9 = box9
  Scene.add(box9);

  const material10 = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  const box10 = new THREE.Mesh(geometry7, material10);
  box10.position.set(-15, 1, 22);
  box10.castShadow = true
  innerObject.box10 = box10
  Scene.add(box10);


 
  // 精灵文本框
  let Sprite1 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: 'Basic 材质',
    text2: 'Sprite 标签'
  }, ((res) => {
    Sprite1 = res
    Sprite1.scale.set(3, 1, 1);
    Sprite1.position.set(-15, 3, 10);
    Scene.add(Sprite1)
  }))

  let Sprite2 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: 'Phong 材质',
    text2: 'Sprite 标签'
  }, ((res) => {
    Sprite2 = res
    Sprite2.scale.set(3, 1, 1);
    Sprite2.position.set(-15, 3, 14);
    Scene.add(Sprite2)
  }))

  let Sprite3 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: 'Lambert 材质',
    text2: 'Sprite 标签'
  }, ((res) => {
    Sprite3 = res
    Sprite3.scale.set(3, 1, 1);
    Sprite3.position.set(-15, 3, 18);
    Scene.add(Sprite3)
  }))

  let Sprite4 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: 'Standard 材质',
    text2: 'Sprite 标签'
  }, ((res) => {
    Sprite4 = res
    Sprite4.scale.set(3, 1, 1);
    Sprite4.position.set(-15, 3, 22);
    Scene.add(Sprite4)
  }))

  let Sprite5 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: '点击变色/恢复'
  }, ((res) => {
    Sprite5 = res
    Sprite5.scale.set(3, 1, 1);
    Sprite5.position.set(5, 5, 5);
    Scene.add(Sprite5)
  }))

  let Sprite6 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: '6边不同贴图'
  }, ((res) => {
    Sprite6 = res
    Sprite6.scale.set(3, 1, 1);
    Sprite6.position.set(-10, 6, 0);
    Scene.add(Sprite6)
  }))

  // 2dcssobject
  init2DTag()

  // 水
  const waterGeometry = new THREE.PlaneGeometry( 300, 300 );
  const Water = new ThreeWater(
    waterGeometry,
    {
      textureWidth: 512,
      textureHeight: 512,
      waterNormals: new THREE.TextureLoader().load('/texture/waternormals.jpg', (texture) => {
        texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
      }),
      alpha: 1.0,
      sunDirection: DirectionalLight.position.clone().normalize(),
      sunColor: 0xffffff,
      waterColor: 0xc1dbfe,
      distortionScale: 13.7,
      fog: Scene.fog
    }
  );
  Water.rotation.x = - Math.PI / 2;
  Water.position.y = -0.2;
  Scene.add( Water );
  innerObject.Water = Water


  // 模型
  new GLTFLoader().load('/models/Soldier.glb', (gltf) => {
    
    gltf.scene.scale.set(3, 3, 3)
    gltf.scene.position.set(20, 0, -20)
    gltf.scene.rotateY(Math.PI / 2)
    gltf.scene.traverse(item => {
      if(item.isMesh) {
        item.castShadow = true
      }
    })


    innerObject.model = gltf
    innerObject.modelAnimationMixer = new THREE.AnimationMixer(innerObject.model.scene)
    innerObject.modelActiveID = 3

    modelActive(innerObject.modelActiveID)
    Scene.add(innerObject.model.scene)
  })

  initParticles('system1', "/texture/xh1.png")
  initParticles('system2', "/texture/xh2.png")
  initParticles('system3', "/texture/xh3.png")

  initEffect()
  initGUI()

  
  const axesHelper = new THREE.AxesHelper(10);
  Scene.add(axesHelper);



  beforeChoose = (() => {
    if (clickMeshArr.length && clickTempMaterial) {
      clickMeshArr[0].object.material = clickTempMaterial.clone()
    }
  })

  afterChoose = (() => {
    if (clickMeshArr.length) {
      clickTempMaterial = clickMeshArr[0].object.material.clone()
      let temp = clickMeshArr[0].object.material.color
      temp.r = ((temp.r * 1.2) > 1) ? 1 : (temp.r * 1.2)
      temp.g = ((temp.g * 1.2) > 1) ? 1 : (temp.g * 1.2)
      temp.b = ((temp.b * 1.2) > 1) ? 1 : (temp.b * 1.2)
    }
  })

  // 可交互的模型
  innerObject.interactive.push(innerObject.box1, innerObject.box2, innerObject.box3)

  innerObject.animate = (() => {
    innerObject.box1.rotation.x += 0.01
    innerObject.box1.rotation.y += 0.01
    innerObject.box2.rotation.x += 0.001
    innerObject.box2.rotation.y += 0.001
    innerObject.box3.rotation.x += 0.005
    innerObject.box3.rotation.y += 0.005
    innerObject.box6.rotation.x += 0.01
    innerObject.box6.rotation.y += 0.01
    Water.material.uniforms.time.value += 0.3 / 60
  })
  animateFlag = innerObject.animate

  window.addEventListener('click', choose); // 监听窗口鼠标单击事件
}

function addSprite(config = {}, cb) {
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  const canvasImage = new Image()
  canvasImage.src = config.imgSrc
  canvasImage.onload = function () {
    canvas.width = canvasImage.width / 3
    canvas.height = canvasImage.height / 3
    context.drawImage(canvasImage, 0, 0, canvasImage.width / 3, canvasImage.height / 3)
    context.font = "Bold 12px sans-serif";
    context.fillStyle = "#05184b";
    context.textAlign = 'center'
    if(config.text1 && config.text2) {
      context.fillText(config.text1, canvasImage.width / 3 / 2, canvasImage.height / 3 * 0.4);
      context.fillText(config.text2, canvasImage.width / 3 / 2, canvasImage.height / 3 * 0.8);
    } else if (config.text1 && !config.text2) {
      context.fillText(config.text1, canvasImage.width / 3 / 2, canvasImage.height / 3 * 0.6);
    }

    const spriteTexture = new THREE.Texture(canvas);
    spriteTexture.needsUpdate = true;

    const spriteMaterial = new THREE.SpriteMaterial({ map: spriteTexture });
    const sprite = new THREE.Sprite(spriteMaterial);
    
    if(cb) {
      cb(sprite)
    }
  }
}

function initEffect() {
  // postprocessing
  composer = new EffectComposer( Renderer );
  composer.addPass( new RenderPass( Scene, Camera ) );
  afterimagePass = new AfterimagePass();
  afterimagePass.uniforms['damp'].value = 0.94
  composer.addPass( afterimagePass );
}


function initFog() {
  if(isFog.value) {
    Scene.fog = new THREE.Fog( 0xc4ddfc, 0.01, 100);
  } else {
    Scene.fog = null
  }
}


function modelActive(id) {
  // 0 立定 1 跑 2 Tpose 3 走
  innerObject.modelActiveID = id
  
  if(innerObject.modelAnimationMixer) {
    if(innerObject.modelAnimationMixer._actions.length) {
      innerObject.modelAnimationMixer._actions.forEach(item => {
        item.enabled = false
      })
    }
    const control = innerObject.modelAnimationMixer.clipAction(innerObject.model.animations[id])
    control.enabled = true
    control.play()

    innerObject.modelMove = (() => {
      const model = innerObject.model.scene
      let speed = 0
      if(innerObject.modelActiveID === 1) {
        speed = 0.22
      } else if (innerObject.modelActiveID === 3) {
        speed = 0.087
      }
      
      if(model.position.x <= -20 && model.position.z <= 20) {
        model.rotation.y = Math.PI
        model.position.z += speed
      } else if (model.position.z >= 20 && model.position.x <= 20) {
        model.rotation.y = -Math.PI / 2
        model.position.x += speed
      } else if (model.position.z >= -20 && model.position.x >= 20) {
        model.rotation.y = 0
        model.position.z -= speed
      } else if (model.position.z <= -20 && model.position.x >=- 20) {
        model.rotation.y = Math.PI / 2
        model.position.x -= speed
      }
    })
  }
}


function init2DTag() {
  const div = document.createElement('div');
  div.innerHTML = 'CSS2DObject 标签';
  div.style.padding = '4px 10px';
  div.style.color = '#fff';
  div.style.fontSize = '12px';
  div.style.position = 'absolute';
  div.style.width = '80px';
  div.style.height = '40px';
  div.style.backgroundColor = 'rgba(25,25,25,0.5)';
  div.style.borderRadius = '5px';
  div.style.pointerEvents = 'all';
  const label = new CSS2DObject(div)
  innerObject.box8.add(label)
  label.position.y = innerObject.box8.position.y + 0.2

  LabelRenderer = new CSS2DRenderer();
  LabelRenderer.setSize( window.innerWidth, window.innerHeight );
  LabelRenderer.domElement.style.position = 'absolute';
  LabelRenderer.domElement.style.top = 0;
  LabelRenderer.domElement.style.left = 0;
  LabelRenderer.domElement.style.pointerEvents = 'none';
  document.body.appendChild( LabelRenderer.domElement );
}

function initParticles(name, path) {
  let geom = new THREE.BufferGeometry()
  let material = new THREE.PointsMaterial({
    size: 2,
    transparent: true,
    opacity: 0.6,
    vertexColors: false,
    map: new THREE.TextureLoader().load(path),
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    sizeAttenuation: true
  })

  let positions = []
  let velocities = []

  let range = 500
  for(let i = 0; i < 10000; i++) {
    positions.push(
      Math.random() * range - range / 2,
      Math.random() * range - 50,
      Math.random() * range - range / 2,
    )

    velocities.push(
      (Math.random() - 0.5) / 3,
      0.1 + Math.random() / 5,
      (Math.random() - 0.5) / 3
    )
  }

  geom.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
  geom.setAttribute('velocity', new THREE.Float32BufferAttribute(velocities, 3))


  let cloud = new THREE.Points(geom, material)
  cloud.name = name
  Scene.add(cloud)
}

function updateSnow() {
  Scene.children.forEach(child => {
    if(child instanceof THREE.Points) {
      let cloud = child
      const pos_BufferAttr = cloud.geometry.getAttribute('position')
      const vel_BufferAttr = cloud.geometry.getAttribute('velocity')

      for(let i = 0; i < pos_BufferAttr.count; i++) {
        let pos_x = pos_BufferAttr.getX(i)
        let pos_y = pos_BufferAttr.getY(i)
        let pos_z = pos_BufferAttr.getZ(i)

        let vel_x = vel_BufferAttr.getX(i)
        let vel_y = vel_BufferAttr.getY(i)
        let vel_z = vel_BufferAttr.getZ(i)

        pos_x = pos_x - vel_x
        pos_y = pos_y - vel_y
        pos_z = pos_z - vel_z


        if(pos_x <= -200 || pos_x >= 200) vel_x = vel_x * -1
        if(pos_y <= 0) pos_y = 600
        if(pos_z <= -200 || pos_z >= 200) vel_z = vel_z * -1

        pos_BufferAttr.setX(i, pos_x)
        pos_BufferAttr.setY(i, pos_y)
        pos_BufferAttr.setZ(i, pos_z)

        vel_BufferAttr.setX(i, vel_x)
        vel_BufferAttr.setZ(i, vel_z)
      }
      pos_BufferAttr.needsUpdate = true
      vel_BufferAttr.needsUpdate = true
    }
  })
}



function animate() {
  requestAnimationFrame(animate)
  Controls.update()
  if (innerObject.modelAnimationMixer) {
    innerObject.modelAnimationMixer.update(clock.getDelta())
  }
  animateFlag()
  if(innerObject.modelMove) {
    innerObject.modelMove()
  }
  if(isSnow.value) {
    updateSnow()
  }
  myStats.update()

  if(isAfterEffect.value) {
    composer.render()
  } else {
    Renderer.render(Scene, Camera)
  }

  LabelRenderer.render( Scene, Camera );
}



function initGUI() {
  const gui = new GUI({ // 可接收参数
    name: '色调映射(线性)',
    width: 300,
    closed: true
  })

  gui.add(Renderer, 'toneMappingExposure', 0, 1, 0.01).name("色调映射(线性)")
}





/**
 * 射线投射器`Raycaster`的射线拾取选中网格模型对象函数choose()
 * 选中的网格模型变为半透明效果
 */
function choose(event) {
  beforeChoose()

  let Sx = event.clientX; //鼠标单击位置横坐标
  let Sy = event.clientY; //鼠标单击位置纵坐标
  //屏幕坐标转WebGL标准设备坐标
  let x = (Sx / windowWidth) * 2 - 1; //WebGL标准设备横坐标
  let y = -(Sy / windowHeight) * 2 + 1; //WebGL标准设备纵坐标
  //创建一个射线投射器`Raycaster`
  let raycaster = new THREE.Raycaster();
  //通过鼠标单击位置标准设备坐标和相机参数计算射线投射器`Raycaster`的射线属性.ray
  raycaster.setFromCamera(new THREE.Vector2(x, y), Camera);
  //返回.intersectObjects()参数中射线选中的网格模型对象
  // 未选中对象返回空数组[],选中一个数组1个元素，选中两个数组两个元素
  let arr = []
  innerObject.interactive.forEach(Mesh => {
    arr.push(Mesh)
  })
  let intersects = raycaster.intersectObjects(arr)
  clickMeshArr = intersects
  

  afterChoose()
}

function handleLight(item) {
  const lightItem = lights.value.find(e => e.name === item.name)
  lightItem.open = !lightItem.open
  switch (item.name) {
    case '环境光':
      AmbientLight.visible = !AmbientLight.visible
      break
    case '平行光':
      DirectionalLight.visible = !DirectionalLight.visible
      break
    case '点光源':
      PointLight.visible = !PointLight.visible
      break
    case '聚光灯':
      SpotLight.visible = !SpotLight.visible
      break
    case '平面光':
      RectAreaLight.visible = !RectAreaLight.visible
      break
  }
}

function handleCamera() {
  isOrthographicCamera.value = !isOrthographicCamera.value
  initCamera()
}

function handleFloor() {
  isMirrorFloor.value = !isMirrorFloor.value
  initFloor()
}

function handleFog() {
  isFog.value = !isFog.value
  initFog()
}

function handleSnow() {
  isSnow.value = !isSnow.value
  if(isSnow.value) {
    initParticles('system1', "/texture/xh1.png")
    initParticles('system2', "/texture/xh2.png")
    initParticles('system3', "/texture/xh3.png")
  } else {
    const point1 = Scene.children.find(e => e.name === 'system1')
    const point2 = Scene.children.find(e => e.name === 'system2')
    const point3 = Scene.children.find(e => e.name === 'system3')
    Scene.remove(point1)
    Scene.remove(point2)
    Scene.remove(point3)
  }
}

function handleEffect() {
  isAfterEffect.value = !isAfterEffect.value
}



window.addEventListener('resize', onWindowResize);
function onWindowResize() {
  Renderer.setSize(window.innerWidth, window.innerHeight);
  Camera.aspect = (window.innerWidth / window.innerHeight);
  Camera.updateProjectionMatrix();
}



onBeforeMount(() => {
  window.removeEventListener('click', choose)
})

</script>

<style scoped>
.lightBtn {
  position: fixed;
  width: 100%;
  top: 10px;
  left: 90px;
  display: flex;
  user-select: none;
}

.lightBtn button {
  color: black;
  margin-left: 10px;
}
</style>