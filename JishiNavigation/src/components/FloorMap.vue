<template>
    <div class="floor-map">
        <div ref="container" class="scene-container"></div>
        <div class="controls">
            <div class="search-box">
                <input v-model="searchQuery" placeholder="搜索房间..." @input="handleSearch" />
            </div>
        </div>
        <div class="room-info" v-if="selectedRoom">
            <h3>{{ selectedRoom.name }}</h3>
            <p>{{ selectedRoom.description }}</p>
        </div>
    </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader'
import roomsData from '@/assets/data/rooms.json'
import { markRaw } from 'vue'

export default {
    name: 'FloorMap',
    data() {
        return {
            selectedRoom: null,
            searchQuery: '',
            _threeObjects: markRaw({
                scene: null,
                camera: null,
                renderer: null,
                controls: null,
                roomMeshes: new Map(),
                animationFrameId: null,
                raycaster: null,
                mouse: null,
                model: null
            })
        }
    },
    mounted() {
        this.$nextTick(() => {
            this.initScene()
            this.loadModel()
            this.setupRaycaster()
            this.startAnimation()
        })
        window.addEventListener('resize', this.onWindowResize)
    },
    beforeUnmount() {
        this.cleanup()
    },
    methods: {
        initScene() {
            const three = this._threeObjects

            // 创建场景
            three.scene = new THREE.Scene()
            three.scene.background = new THREE.Color(0xf0f0f0)

            // 创建相机
            three.camera = new THREE.PerspectiveCamera(
                60,
                window.innerWidth / window.innerHeight,
                0.1,
                1000
            )
            three.camera.position.set(50, 50, 50)
            three.camera.lookAt(0, 0, 0)

            // 创建渲染器
            three.renderer = new THREE.WebGLRenderer({
                antialias: true,
                alpha: true
            })
            three.renderer.setSize(window.innerWidth, window.innerHeight)
            three.renderer.setPixelRatio(window.devicePixelRatio)
            three.renderer.shadowMap.enabled = true
            three.renderer.shadowMap.type = THREE.PCFSoftShadowMap
            this.$refs.container.appendChild(three.renderer.domElement)

            // 添加控制器
            three.controls = new OrbitControls(three.camera, three.renderer.domElement)
            three.controls.enableDamping = true
            three.controls.dampingFactor = 0.05
            three.controls.maxPolarAngle = Math.PI / 2
            three.controls.minDistance = 20
            three.controls.maxDistance = 150

            // 添加光源
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
            three.scene.add(ambientLight)

            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
            directionalLight.position.set(50, 50, 50)
            directionalLight.castShadow = true
            directionalLight.shadow.mapSize.width = 2048
            directionalLight.shadow.mapSize.height = 2048
            three.scene.add(directionalLight)
        },

        loadModel() {
            const three = this._threeObjects

            // 创建DRACOLoader实例
            const dracoLoader = new DRACOLoader()
            dracoLoader.setDecoderPath('/draco/') // 设置DRACO解码器路径

            // 创建GLTFLoader实例
            const loader = new GLTFLoader()
            loader.setDRACOLoader(dracoLoader)

            // 加载模型
            loader.load(
                '/models/model.glb',
                (gltf) => {
                    three.model = gltf.scene
                    three.model.scale.set(1, 1, 1) // 根据需要调整模型大小
                    three.model.position.set(0, 0, 0) // 根据需要调整模型位置
                    three.scene.add(three.model)

                    // 遍历模型中的所有网格
                    gltf.scene.traverse((child) => {
                        if (child.isMesh) {
                            child.castShadow = true
                            child.receiveShadow = true
                        }
                    })
                },
                (xhr) => {
                    console.log((xhr.loaded / xhr.total * 100) + '% loaded')
                },
                (error) => {
                    console.error('An error happened:', error)
                }
            )
        },

        loadRooms() {
            const three = this._threeObjects

            roomsData.rooms.forEach(roomData => {
                const geometry = new THREE.BoxGeometry(
                    roomData.dimensions.width,
                    roomData.dimensions.height,
                    roomData.dimensions.depth
                )

                const material = new THREE.MeshStandardMaterial({
                    color: roomData.type === 'office' ? 0x88ccff : 0x90caf9,
                    roughness: 0.7,
                    metalness: 0.3,
                    transparent: true,
                    opacity: 0.9
                })

                const mesh = new THREE.Mesh(geometry, material)
                mesh.position.set(
                    roomData.position.x,
                    roomData.position.y + roomData.dimensions.height / 2,
                    roomData.position.z
                )

                // 添加房间编号
                const textGeometry = new THREE.BoxGeometry(0.5, 2, 0.1)
                const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 })
                const textMesh = new THREE.Mesh(textGeometry, textMaterial)
                textMesh.position.set(0, 0, roomData.dimensions.depth / 2 + 0.1)
                mesh.add(textMesh)

                mesh.castShadow = true
                mesh.receiveShadow = true
                mesh.userData.roomData = roomData
                three.roomMeshes.set(roomData.id, mesh)
                three.scene.add(mesh)
            })
        },

        createCorridors() {
            const three = this._threeObjects

            roomsData.corridors.forEach(corridor => {
                // 创建走廊地面
                const points = corridor.path
                const width = corridor.width

                for (let i = 0; i < points.length - 1; i++) {
                    const start = points[i]
                    const end = points[i + 1]

                    // 计算走廊段的长度和方向
                    const length = new THREE.Vector3(
                        end.x - start.x,
                        end.y - start.y,
                        end.z - start.z
                    ).length()

                    const geometry = new THREE.BoxGeometry(width, 0.1, length)
                    const material = new THREE.MeshStandardMaterial({
                        color: 0xbdbdbd,
                        roughness: 0.8,
                        metalness: 0.2
                    })

                    const corridorMesh = new THREE.Mesh(geometry, material)

                    // 设置走廊位置和旋转
                    corridorMesh.position.set(
                        (start.x + end.x) / 2,
                        0.05,
                        (start.z + end.z) / 2
                    )

                    // 计算旋转角度
                    const angle = Math.atan2(end.z - start.z, end.x - start.x)
                    corridorMesh.rotation.y = angle

                    three.scene.add(corridorMesh)
                }
            })
        },

        createAreas() {
            const three = this._threeObjects

            roomsData.areas.forEach(area => {
                const geometry = new THREE.PlaneGeometry(
                    area.dimensions.width,
                    area.dimensions.depth
                )
                const material = new THREE.MeshStandardMaterial({
                    color: 0xb3e5fc,
                    transparent: true,
                    opacity: 0.3,
                    side: THREE.DoubleSide
                })

                const mesh = new THREE.Mesh(geometry, material)
                mesh.position.set(
                    area.position.x,
                    area.dimensions.height,
                    area.position.z
                )
                mesh.rotation.x = -Math.PI / 2

                three.scene.add(mesh)
            })
        },

        setupRaycaster() {
            const three = this._threeObjects
            three.raycaster = new THREE.Raycaster()
            three.mouse = new THREE.Vector2()

            const container = this.$refs.container

            const onMouseMove = (event) => {
                three.mouse.x = (event.clientX / window.innerWidth) * 2 - 1
                three.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1

                three.raycaster.setFromCamera(three.mouse, three.camera)
                const intersects = three.raycaster.intersectObjects(three.scene.children)

                three.roomMeshes.forEach(mesh => {
                    if (mesh.material) {
                        mesh.material.color.setHex(
                            mesh.userData.roomData.type === 'office' ? 0x88ccff : 0x90caf9
                        )
                    }
                })

                for (const intersect of intersects) {
                    if (intersect.object.userData.roomData) {
                        intersect.object.material.color.setHex(0xff8888)
                        break
                    }
                }
            }

            const onClick = (event) => {
                three.mouse.x = (event.clientX / window.innerWidth) * 2 - 1
                three.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1

                three.raycaster.setFromCamera(three.mouse, three.camera)
                const intersects = three.raycaster.intersectObjects(three.scene.children)

                for (const intersect of intersects) {
                    if (intersect.object.userData.roomData) {
                        this.selectedRoom = { ...intersect.object.userData.roomData }
                        break
                    }
                }
            }

            container.addEventListener('mousemove', onMouseMove)
            container.addEventListener('click', onClick)

            // 保存事件监听器引用以便清理
            three.eventListeners = {
                mousemove: onMouseMove,
                click: onClick
            }
        },

        startAnimation() {
            const three = this._threeObjects

            const animate = () => {
                three.animationFrameId = requestAnimationFrame(animate)

                if (three.controls) {
                    three.controls.update()
                }

                if (three.renderer && three.scene && three.camera) {
                    three.renderer.render(three.scene, three.camera)
                }
            }

            animate()
        },

        cleanup() {
            const three = this._threeObjects

            // 清理动画
            if (three.animationFrameId !== null) {
                cancelAnimationFrame(three.animationFrameId)
            }

            // 清理事件监听器
            window.removeEventListener('resize', this.onWindowResize)
            if (three.eventListeners) {
                const container = this.$refs.container
                container.removeEventListener('mousemove', three.eventListeners.mousemove)
                container.removeEventListener('click', three.eventListeners.click)
            }

            // 清理渲染器
            if (three.renderer) {
                three.renderer.dispose()
                three.renderer.forceContextLoss()
                three.renderer.domElement.remove()
            }

            // 清理几何体和材质
            three.roomMeshes.forEach(mesh => {
                mesh.geometry.dispose()
                mesh.material.dispose()
            })

            // 清理场景
            if (three.scene) {
                three.scene.clear()
            }

            // 重置所有Three.js对象
            Object.keys(three).forEach(key => {
                three[key] = null
            })
        },

        onWindowResize() {
            const three = this._threeObjects

            if (three.camera && three.renderer) {
                three.camera.aspect = window.innerWidth / window.innerHeight
                three.camera.updateProjectionMatrix()
                three.renderer.setSize(window.innerWidth, window.innerHeight)
            }
        },

        handleSearch() {
            const three = this._threeObjects

            three.roomMeshes.forEach(mesh => {
                mesh.material.color.setHex(
                    mesh.userData.roomData.type === 'office' ? 0x88ccff : 0x90caf9
                )
            })

            if (this.searchQuery) {
                const query = this.searchQuery.toLowerCase()
                three.roomMeshes.forEach(mesh => {
                    const roomData = mesh.userData.roomData
                    if (
                        roomData.name.toLowerCase().includes(query) ||
                        roomData.description.toLowerCase().includes(query)
                    ) {
                        mesh.material.color.setHex(0xff8888)
                    }
                })
            }
        }
    }
}
</script>

<style scoped>
.floor-map {
    width: 100vw;
    height: 100vh;
    position: relative;
}

.scene-container {
    width: 100%;
    height: 100%;
}

.controls {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 10;
}

.search-box input {
    padding: 8px 12px;
    border: none;
    border-radius: 4px;
    background: rgba(255, 255, 255, 0.9);
    width: 200px;
}

.room-info {
    position: absolute;
    right: 20px;
    top: 20px;
    background: rgba(255, 255, 255, 0.9);
    padding: 20px;
    border-radius: 8px;
    max-width: 300px;
}
</style>