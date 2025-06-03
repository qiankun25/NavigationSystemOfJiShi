<template>
    <div class="floor-map">
        <div ref="container" class="scene-container"></div>
        <div class="room-buttons">
            <button v-for="room in rooms" :key="room.id" class="room-button"
                :class="{ 'active': selectedRoom?.id === room.id }" @click="selectRoom(room)">
                {{ room.id }}
            </button>
        </div>
        <div class="sidebar" v-if="selectedRoom">
            <div class="room-info">
                <div class="room-image" v-if="selectedRoom.image">
                    <img :src="selectedRoom.image" :alt="selectedRoom.name">
                </div>
                <h3>{{ selectedRoom.name }}</h3>
                <div class="info-item">
                    <label>房间号：</label>
                    <span>{{ selectedRoom.id }}</span>
                </div>
                <div class="info-item">
                    <label>类型：</label>
                    <span>{{ selectedRoom.type === 'office' ? '办公室' : '实验室' }}</span>
                </div>
                <div class="info-item">
                    <label>描述：</label>
                    <p>{{ selectedRoom.description }}</p>
                </div>
                <div class="info-item" v-if="selectedRoom.teachers && selectedRoom.teachers.length">
                    <label>教师：</label>
                    <p>{{ selectedRoom.teachers.join('、') }}</p>
                </div>
            </div>
        </div>
        <div class="controls">
            <div class="search-box">
                <input v-model="searchQuery" placeholder="搜索房间..." @input="handleSearch" />
            </div>
        </div>
        <div class="loading" v-if="isLoading">
            <div class="loading-text">加载中... {{ loadingProgress }}%</div>
        </div>
        <div class="error-message" v-if="errorMessage">
            {{ errorMessage }}
        </div>
        <div class="controls-tips">
            <h3>控制提示：</h3>
            <ul>
                <li>左键拖动：旋转视角</li>
                <li>右键拖动：平移视角</li>
                <li>滚轮：缩放</li>
                <li>双击：重置视角</li>
            </ul>
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
import gsap from 'gsap'

export default {
    name: 'FloorMap',
    data() {
        return {
            selectedRoom: null,
            searchQuery: '',
            isLoading: true,
            loadingProgress: 0,
            errorMessage: '',
            rooms: [],
            _threeObjects: markRaw({
                scene: null,
                camera: null,
                renderer: null,
                controls: null,
                roomMeshes: new Map(),
                animationFrameId: null,
                raycaster: null,
                mouse: null,
                model: null,
                defaultCameraPosition: new THREE.Vector3(5, 5, 5),
                searchLight: null, // 搜索提示光源
                lightAnimation: null // 光源动画
            })
        }
    },
    mounted() {
        this.$nextTick(() => {
            this.initScene()
            this.loadModel()
            this.setupRaycaster()
            this.startAnimation()
            // 加载房间数据
            this.loadRoomsData()
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
                75,
                window.innerWidth / window.innerHeight,
                0.1,
                1000
            )
            three.camera.position.copy(three.defaultCameraPosition)
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

            // 添加并配置 OrbitControls
            three.controls = new OrbitControls(three.camera, three.renderer.domElement)
            three.controls.enableDamping = true // 启用阻尼效果
            three.controls.dampingFactor = 0.05 // 阻尼系数
            three.controls.rotateSpeed = 0.5 // 旋转速度
            three.controls.zoomSpeed = 1.2 // 缩放速度
            three.controls.panSpeed = 0.8 // 平移速度
            three.controls.minDistance = 2 // 最小缩放距离
            three.controls.maxDistance = 20 // 最大缩放距离
            three.controls.maxPolarAngle = Math.PI / 1.5 // 限制垂直旋转角度
            three.controls.enablePan = true // 启用平移
            three.controls.screenSpacePanning = true // 使平移始终平行于屏幕

            // 添加双击重置功能
            this.$refs.container.addEventListener('dblclick', this.resetCamera)

            // 添加光源
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
            three.scene.add(ambientLight)

            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
            directionalLight.position.set(10, 10, 10)
            directionalLight.castShadow = true
            three.scene.add(directionalLight)

            // 添加辅助网格
            const gridHelper = new THREE.GridHelper(10, 10)
            three.scene.add(gridHelper)

            // 添加坐标轴辅助
            const axesHelper = new THREE.AxesHelper(5)
            three.scene.add(axesHelper)
        },

        loadModel() {
            const three = this._threeObjects

            // 创建DRACOLoader实例
            const dracoLoader = new DRACOLoader()
            dracoLoader.setDecoderPath('/draco/')

            // 创建GLTFLoader实例
            const loader = new GLTFLoader()
            loader.setDRACOLoader(dracoLoader)

            // 加载模型
            loader.load(
                '/models/model.glb',
                (gltf) => {
                    console.log('Model loaded successfully:', gltf)
                    three.model = gltf.scene

                    // 遍历模型中的所有子网格
                    gltf.scene.traverse((child) => {
                        if (child.isMesh && child.name) {
                            console.log('Found mesh:', child.name, child.position)
                            // 存储房间网格引用
                            three.roomMeshes.set(child.name, child)
                        }
                    })

                    // 自动调整模型大小和位置
                    const box = new THREE.Box3().setFromObject(gltf.scene)
                    const center = box.getCenter(new THREE.Vector3())
                    const size = box.getSize(new THREE.Vector3())

                    // 计算适当的缩放比例
                    const maxDim = Math.max(size.x, size.y, size.z)
                    const scale = 5 / maxDim
                    gltf.scene.scale.multiplyScalar(scale)

                    // 将模型居中
                    gltf.scene.position.sub(center.multiplyScalar(scale))

                    three.scene.add(gltf.scene)

                    // 调整相机位置以适应模型
                    const distance = Math.max(size.x, size.y, size.z) * 2
                    three.camera.position.set(distance, distance, distance)
                    three.camera.lookAt(0, 0, 0)

                    this.isLoading = false
                    this.loadingProgress = 100
                },
                (xhr) => {
                    this.loadingProgress = Math.round((xhr.loaded / xhr.total) * 100)
                    console.log(this.loadingProgress + '% loaded')
                },
                (error) => {
                    console.error('Error loading model:', error)
                    this.errorMessage = '模型加载失败: ' + error.message
                    this.isLoading = false
                }
            )
        },

        loadRoomsData() {
            // 从rooms.json加载房间数据
            this.rooms = roomsData.rooms
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
            }

            const onClick = (event) => {
                three.mouse.x = (event.clientX / window.innerWidth) * 2 - 1
                three.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1

                three.raycaster.setFromCamera(three.mouse, three.camera)
                const intersects = three.raycaster.intersectObjects(three.scene.children)

                for (const intersect of intersects) {
                    if (intersect.object.userData.roomData) {
                        this.selectRoom(intersect.object.userData.roomData)
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

            // 移除双击事件监听器
            if (this.$refs.container) {
                this.$refs.container.removeEventListener('dblclick', this.resetCamera)
            }

            // 移除搜索光源
            this.removeSearchLight()
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
            const query = this.searchQuery.toLowerCase().trim()

            // 移除之前的搜索光源
            this.removeSearchLight()

            if (!query) {
                return
            }

            // 在rooms数据中搜索
            const foundRoom = this.rooms.find(room =>
                room.id.toLowerCase().includes(query) ||
                room.name.toLowerCase().includes(query) ||
                room.description.toLowerCase().includes(query) ||
                (room.teachers && room.teachers.some(teacher =>
                    teacher.toLowerCase().includes(query)
                ))
            )

            if (foundRoom) {
                // 输出找到的房间信息
                console.log('找到房间:', {
                    id: foundRoom.id,
                    name: foundRoom.name,
                    type: foundRoom.type
                })

                // 更新选中的房间
                this.selectedRoom = foundRoom

                // 在3D模型中查找对应的房间mesh
                let roomMesh = null
                if (three.model) {
                    three.model.traverse((child) => {
                        if (child.isMesh && child.name) {
                            // 输出所有mesh的名字，帮助调试
                            console.log('遍历到Mesh:', child.name)

                            if (child.name === foundRoom.id) {
                                roomMesh = child
                                const worldPosition = child.getWorldPosition(new THREE.Vector3())
                                // 输出找到的mesh信息
                                console.log('找到对应Mesh:', {
                                    name: child.name,
                                    position: child.position,
                                    worldPosition: worldPosition
                                })
                                // 在房间位置添加闪烁的点光源
                                this.addSearchLight(worldPosition)
                            }
                        }
                    })

                    if (!roomMesh) {
                        console.warn('未找到房间对应的Mesh:', foundRoom.id)
                    }
                } else {
                    console.warn('3D模型未加载')
                }

                if (roomMesh) {
                    // 移动相机到房间位置
                    const worldPosition = new THREE.Vector3()
                    roomMesh.getWorldPosition(worldPosition)
                    this.moveCameraToTarget(worldPosition)
                }
            } else {
                // 没有找到匹配的房间
                this.showNoResultsMessage()
            }
        },

        resetSearchState() {
            // 清除错误消息
            this.errorMessage = ''
        },

        showNoResultsMessage() {
            this.errorMessage = '未找到匹配的房间'
            // 3秒后自动清除错误消息
            setTimeout(() => {
                this.errorMessage = ''
            }, 3000)
        },

        selectRoom(room) {
            this.selectedRoom = room

            const three = this._threeObjects;
            // 移除之前的搜索光源
            this.removeSearchLight();

            if (three.model) {
                let roomMesh = null;
                three.model.traverse((child) => {
                    if (child.isMesh && child.name === room.id) {
                        roomMesh = child;
                    }
                });

                if (roomMesh) {
                    const worldPosition = new THREE.Vector3();
                    roomMesh.getWorldPosition(worldPosition);

                    // 添加闪烁光源
                    this.addSearchLight(worldPosition);
                    // 平滑移动相机
                    this.moveCameraToTarget(worldPosition);
                } else {
                    console.warn('未找到房间对应的 Mesh:', room.id);
                }
            } else {
                console.warn('3D 模型尚未加载完成');
            }

        },

        focusOnRoom(roomMesh) {
            const three = this._threeObjects
            const position = roomMesh.position.clone()
            position.y += 5 // 稍微抬高视角
            position.z += 5 // 向后移动一些

            // 使用GSAP动画平滑移动相机
            gsap.to(three.camera.position, {
                x: position.x,
                y: position.y,
                z: position.z,
                duration: 1,
                ease: 'power2.inOut'
            })

            // 让相机看向房间中心
            three.controls.target.copy(roomMesh.position)
            three.controls.update()
        },

        resetHighlight() {
            const three = this._threeObjects
            if (three.highlightedRoom && three.originalColors.has(three.highlightedRoom.name)) {
                // 恢复原始颜色
                three.highlightedRoom.material.color.copy(three.originalColors.get(three.highlightedRoom.name))
                three.highlightedRoom = null
            }
        },

        moveCameraToTarget(targetPosition) {
            const three = this._threeObjects
            const camera = three.camera
            const controls = three.controls

            // 使用GSAP动画平滑移动相机
            gsap.to(camera.position, {
                x: targetPosition.x + 0.5,
                y: targetPosition.y + 2,
                z: targetPosition.z + 0.5,
                duration: 1,
                ease: 'power2.inOut'
            })

            // 让相机看向目标位置
            controls.target.copy(targetPosition)
            controls.update()
        },

        addSearchLight(position) {
            const three = this._threeObjects


            // 创建点光源
            const light = new THREE.PointLight(0xff0000, 2, 3)
            light.position.set(position.x + 0.2, position.y + 0.5, position.z)
            three.scene.add(light)
            three.searchLight = light

            // 创建光源辅助球体（让光源可见）
            const sphereGeometry = new THREE.SphereGeometry(0.05, 16, 16)
            const sphereMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 })
            const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial)
            light.add(sphere)

            // 使用GSAP创建闪烁动画
            three.lightAnimation = gsap.to(light, {
                intensity: 0.5,
                duration: 0.8,
                repeat: -1,
                yoyo: true,
                ease: "power1.inOut"
            })
        },

        removeSearchLight() {
            const three = this._threeObjects
            if (three.searchLight) {
                // 停止动画
                if (three.lightAnimation) {
                    three.lightAnimation.kill()
                    three.lightAnimation = null
                }
                // 移除光源
                three.scene.remove(three.searchLight)
                three.searchLight = null
            }
        }
    }
}
</script>

<style scoped>
.floor-map {
    position: relative;
    width: 100%;
    height: 100vh;
}

.scene-container {
    width: 100%;
    height: 100%;
}

.room-buttons {
    position: absolute;
    top: 70px;
    left: 20px;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    max-width: 80%;
    background: rgba(255, 255, 255, 0.95);
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    z-index: 100;
}

.room-button {
    padding: 8px 16px;
    background: #ffffff;
    border: 1px solid #e0e0e0;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.3s ease;
}

.room-button:hover {
    background: #f5f5f5;
    transform: translateY(-2px);
}

.room-button.active {
    background: #2196f3;
    color: white;
    border-color: #1976d2;
}

.room-image {
    width: 100%;
    height: 200px;
    margin-bottom: 15px;
    border-radius: 8px;
    overflow: hidden;
}

.room-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.sidebar {
    position: absolute;
    right: 0;
    top: 0;
    width: 300px;
    height: 100%;
    background: rgba(255, 255, 255, 0.95);
    padding: 20px;
    box-shadow: -2px 0 8px rgba(0, 0, 0, 0.1);
    overflow-y: auto;
    z-index: 100;
}

.room-info {
    margin-top: 20px;
}

.room-info h3 {
    font-size: 1.5em;
    margin-bottom: 20px;
    color: #333;
    border-bottom: 2px solid #2196f3;
    padding-bottom: 10px;
}

.info-item {
    margin-bottom: 15px;
    background: rgba(255, 255, 255, 0.8);
    padding: 10px;
    border-radius: 4px;
}

.info-item label {
    font-weight: bold;
    color: #666;
    display: block;
    margin-bottom: 5px;
}

.info-item p {
    margin: 5px 0;
    color: #333;
    line-height: 1.4;
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

.loading {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 20px;
    border-radius: 8px;
    text-align: center;
}

.loading-text {
    font-size: 18px;
    margin-bottom: 10px;
}

.error-message {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(255, 0, 0, 0.8);
    color: white;
    padding: 10px 20px;
    border-radius: 4px;
    z-index: 1000;
    animation: fadeInOut 3s ease-in-out;
}

@keyframes fadeInOut {
    0% {
        opacity: 0;
    }

    10% {
        opacity: 1;
    }

    90% {
        opacity: 1;
    }

    100% {
        opacity: 0;
    }
}

.controls-tips {
    position: absolute;
    bottom: 20px;
    left: 20px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 15px 20px;
    border-radius: 8px;
    font-size: 14px;
    z-index: 1000;
}

.controls-tips h3 {
    margin: 0 0 10px 0;
    font-size: 16px;
}

.controls-tips ul {
    margin: 0;
    padding-left: 20px;
}

.controls-tips li {
    margin: 5px 0;
}
</style>