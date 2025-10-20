<script setup>
import { Application, Assets, Container, Sprite, Texture } from 'pixi.js'
import { onMounted, onUnmounted, getCurrentInstance } from 'vue'
import bunny from '@/assets/bunny.png'

const { proxy } = getCurrentInstance()

let app
let playerSprite
let sceneContainer
const step = 16
const impulseStrength = 1.5 // 碰撞瞬时加速度（更低）
const damping = 0.9 // 速度阻尼（略快衰减）
const minSpeed = 0.03 // 速度阈值（小于则归零）
const maxSpeed = 3 // 最大速度上限（更低）
// 动态边界：以玩家为中心，按屏幕尺寸计算（见 ticker）
const restitution = 0 // 边界反弹系数（0 表示不反弹，直接贴边停下）
const blocks = []

const clamp = (val, min, max) => Math.min(max, Math.max(min, val))

const updateCamera = () => {
    if (!app || !sceneContainer || !playerSprite) return
    sceneContainer.pivot.set(playerSprite.x, playerSprite.y)
    sceneContainer.position.set(app.screen.width / 2, app.screen.height / 2)
}

const createBlock = (x, y, w, h, tint = 0x4aa3ff) => {
    const block = new Sprite(Texture.WHITE)
    block.tint = tint
    block.width = w
    block.height = h
    block.anchor.set(0)
    block.x = x
    block.y = y
    block.vx = 0
    block.vy = 0
    sceneContainer.addChild(block)
    blocks.push(block)
    return block
}

const rectsIntersect = (a, b) =>
    a.x < b.x + b.width && a.x + a.width > b.x && a.y < b.y + b.height && a.y + a.height > b.y

const handleKeyDown = (e) => {
    if (!playerSprite) return
    if (e.repeat) return
    const k = e.key.toLowerCase()
    let dx = 0
    let dy = 0
    if (k === 'w') dy = -step
    else if (k === 's') dy = step
    else if (k === 'a') dx = -step
    else if (k === 'd') dx = step
    else return

    playerSprite.x += dx
    playerSprite.y += dy

    // 碰撞检测：命中则给方块一个速度脉冲（起步快）
    const len = Math.hypot(dx, dy) || 1
    const ux = dx / len
    const uy = dy / len
    const pBounds = playerSprite.getBounds()
    for (const b of blocks) {
        const bBounds = b.getBounds()
        if (rectsIntersect(pBounds, bBounds)) {
            b.vx += ux * impulseStrength
            b.vy += uy * impulseStrength
        }
    }

    updateCamera()
}

onMounted(() => {
    app = new Application()
    const mountEl = proxy.$refs.container
    const { width, height } = mountEl.getBoundingClientRect()
    app.init({
        width,
        height,
    })
        .then(async () => {
            sceneContainer = new Container()
            const texture = await Assets.load(bunny)
            playerSprite = new Sprite(texture)
            playerSprite.anchor.set(0.5)
            playerSprite.x = width / 2
            playerSprite.y = height / 2
            sceneContainer.addChild(playerSprite)
            // 添加一些方块
            createBlock(50, 50, 40, 40, 0xff6b6b)
            createBlock(180, 80, 60, 30, 0x51cf66)
            createBlock(120, 180, 30, 60, 0x845ef7)
            app.stage.addChild(sceneContainer)
            app.start()
            mountEl.appendChild(app.canvas)
            updateCamera()

            window.addEventListener('keydown', handleKeyDown)

            // 每帧更新方块速度衰减与位置
            app.ticker.add((delta) => {
                for (const b of blocks) {
                    if (!b.vx && !b.vy) continue
                    // 速度上限
                    const s = Math.hypot(b.vx, b.vy)
                    if (s > maxSpeed) {
                        b.vx = (b.vx / s) * maxSpeed
                        b.vy = (b.vy / s) * maxSpeed
                    }

                    // 位置更新
                    b.x += b.vx * delta
                    b.y += b.vy * delta

                    // 动态边界：以玩家为中心，保持在屏幕大小的 0.9 倍内
                    const margin = 0.45
                    const minX = playerSprite.x - app.screen.width * margin
                    const maxX = playerSprite.x + app.screen.width * margin - b.width
                    const minY = playerSprite.y - app.screen.height * margin
                    const maxY = playerSprite.y + app.screen.height * margin - b.height

                    if (b.x < minX) {
                        b.x = minX
                        b.vx = 0
                    } else if (b.x > maxX) {
                        b.x = maxX
                        b.vx = 0
                    }
                    if (b.y < minY) {
                        b.y = minY
                        b.vy = 0
                    } else if (b.y > maxY) {
                        b.y = maxY
                        b.vy = 0
                    }

                    // 阻尼
                    b.vx *= damping
                    b.vy *= damping
                    if (Math.abs(b.vx) < minSpeed) b.vx = 0
                    if (Math.abs(b.vy) < minSpeed) b.vy = 0
                }
            })
        })
        .catch((err) => {
            console.error(err)
        })
})

onUnmounted(() => {
    window.removeEventListener('keydown', handleKeyDown)
    if (app) {
        app.destroy(true)
        app = undefined
    }
    playerSprite = undefined
})
</script>

<template>
    <div ref="container" class="container"></div>
</template>

<style scoped lang="less">
.container {
    width: 90vw;
    height: 90vh;
    background-color: #f0f0f0;
}
</style>
