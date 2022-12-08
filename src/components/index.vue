<template>
  <div id="canvasContent"></div>
  <div class="lightBtn">
    <button v-for="item in lights" @click.stop="handleLight(item)">{{ item.name }} {{ item.open ? '已开' : '已关'
    }}</button>
  </div>
  <div class="lightBtn" style="margin-top: 30px">
    <button @click.stop="handleCamera">正交摄像机(与平行光阴影范围同步) {{ isOrthographicCamera ? '已开' : '已关' }}</button>
  </div>
</template>

<script setup>
import { onBeforeMount, onMounted, watch, ref, reactive } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { RectAreaLightUniformsLib } from 'three/examples/jsm/lights/RectAreaLightUniformsLib';
import { RectAreaLightHelper } from 'three/examples/jsm/helpers/RectAreaLightHelper';

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


let Camera = null,
  Scene = null,
  Controls = null,
  Renderer = null,
  AmbientLight = null,
  DirectionalLight = null,
  PointLight = null,
  SpotLight = null,
  RectAreaLight = null,
  isOrthographicCamera = ref(false),
  isMirrorFloor = ref(false),

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
  demo1()

  animate()

  // Controls.addEventListener('end', () => {
  //   console.log(Camera)
  // })
})


function initRenderer() {
  Renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  });
  Renderer.shadowMap.enabled = true
  Renderer.shadowMap.type = THREE.PCFSoftShadowMap
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

}

function initControls() {
  Controls = new OrbitControls(Camera, Renderer.domElement)
  Controls.autoRotate = true
  Controls.autoRotateSpeed = 0.5
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
  const planeGeometry = new THREE.PlaneGeometry(100, 100)
  const planeMaterial = new THREE.MeshLambertMaterial({ color: 0xdddddd })
  const plane = new THREE.Mesh(planeGeometry, planeMaterial)
  plane.receiveShadow = true
  plane.rotateX(-Math.PI / 2)
  innerObject.plane = plane
  Scene.add(innerObject.plane)


  // const planeGeometry = new THREE.PlaneGeometry(100, 100)
  // const planeMaterial = new THREE.MeshLambertMaterial({ color: 0xdddddd })
  // const plane = new THREE.Mesh(planeGeometry, planeMaterial)
  // plane.receiveShadow = true
  // plane.rotateX(-Math.PI / 2)
  // innerObject.plane = plane
  // Scene.add(innerObject.plane)
}

function demo1() {
  innerObject = {}
  innerObject.interactive = []
  animateFlag = null
  Camera.position.set(20, 8, 20)

  initFloor()
  

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

  const geometry4 = new THREE.BoxGeometry(20, 0.1, 20);
  const material4 = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.3, metalness: 0 });
  const floor = new THREE.Mesh(geometry4, material4);
  floor.position.set(0, 0, -30)
  innerObject.floor = floor
  Scene.add(floor);

  const geometry5 = new THREE.BoxGeometry(2, 2, 2);
  const material5 = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0, metalness: 0 });
  const box5 = new THREE.Mesh(geometry5, material5);
  box5.position.set(0, 1, -30);
  innerObject.box5 = box5
  Scene.add(box5);

  const geometry6 = new THREE.BoxBufferGeometry(3, 3, 3)
  const box6ImgSrcArr = [
    "/texture/1.png",
    "/texture/2.png",
    "/texture/3.png",
    "/texture/4.png",
    "/texture/5.png",
    "/texture/6.png"
  ]
  // 创建一组材质， 每个材质对应立方体每个面所用到的材质
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
  })
  animateFlag = innerObject.animate



  let Sprite1 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: 'Basic',
    text2: 'Sprite'
  }, ((res) => {
    Sprite1 = res
    Sprite1.scale.set(3, 1, 1);
    Sprite1.position.set(-15, 3, 10);
    Scene.add(Sprite1)
  }))

  let Sprite2 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: '点击变色/恢复'
  }, ((res) => {
    Sprite2 = res
    Sprite2.scale.set(3, 1, 1);
    Sprite2.position.set(5, 5, 5);
    Scene.add(Sprite2)
  }))

  let Sprite3 = null
  addSprite({
    imgSrc: '/images/1.png',
    text1: '6边不同材质'
  }, ((res) => {
    Sprite3 = res
    Sprite3.scale.set(3, 1, 1);
    Sprite3.position.set(-10, 6, 0);
    Scene.add(Sprite3)
  }))




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
    context.fillStyle = "#81161f";
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


function animate() {
  requestAnimationFrame(animate)
  Controls.update()
  animateFlag()
  Renderer.render(Scene, Camera)
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
  console.log('intersects: ', intersects);

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
  console.log('isOrthographicCamera: ', isOrthographicCamera.value);
  initCamera()
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
  top: 20px;
  left: 20px;
  display: flex;

}

.lightBtn button {
  color: black;
  margin-left: 10px;
}
</style>