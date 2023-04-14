---
title: Go-OpenGL
tags:
  - OpenGL
  - 图形学
categories:
  - 图形学
abbrlink: 7264fd3f
date: 2022-09-14 17:11:24
---

## Intro

OpenGL 是当今最广泛的开源图形库，提供了用于渲染2D、3D矢量图形的跨语言、跨平台的应用程序编程接口（API）。对于图形学基础初学者来说，该库确实是让编程人员与显卡直接交互的最好桥梁。由于本人C++水平不堪，又反感于弱类型的脚本语言，只好退而求其次选择各方面封账都较为完善的Go语言库——`https://github.com/go-gl/gl` 及 `https://github.com/go-gl/glfw`。

![](https://pic1.zhimg.com/v2-d315ea26866b2ea204e06f3f73787dc5_qhd.jpg?source=57bbeac9)

## Installation

```bash
# 安装OpenGL
$ go install github.com/go-gl/gl/v4.1-core/gl@latest
# 安装GLFW应用框架
$ go install github.com/go-gl/glfw/v3.3/glfw@latest
```

>   GLFW 主要用来处理特定操作系统下的特定任务，例如 OpenGL 窗口管理、分辨率切换、键盘、鼠标以及游戏手柄、定时器输入、线程创建等等。

### OpenGL ES

针对手机、PDA和游戏主机等嵌入式设备而设计。

If your target system only provides an OpenGL ES implementation (true for some ARM boards), you need to link against that implementation. You do this by defining the appropriate build tags, e.g.

```
go get -u -tags=gles2 github.com/go-gl/glfw/v3.3/glfw
```



## Sample

[简单样例](https://linux.cn/article-8933-1.html)     [Github](https://github.com/KyleBanks/conways-gol)

```go
package main
import (
    "fmt"
    "log"
    "math/rand"
    "runtime"
    "strings"
    "time"
    "github.com/go-gl/gl/v4.1-core/gl" // OR: github.com/go-gl/gl/v2.1/gl
    "github.com/go-gl/glfw/v3.3/glfw"
)
const (
    width  = 500
    height = 500
    vertexShaderSource = `
        #version 410
        in vec3 vp;
        void main() {
            gl_Position = vec4(vp, 1.0);
        }
    ` + "\x00"
    fragmentShaderSource = `
        #version 410
        out vec4 frag_colour;
        void main() {
            frag_colour = vec4(1, 1, 1, 1.0);
        }
    ` + "\x00"
    rows    = 100
    columns = 100
    threshold = 0.15
    fps       = 10
)
var (
    square = []float32{
        -0.5, 0.5, 0,
        -0.5, -0.5, 0,
        0.5, -0.5, 0,
        -0.5, 0.5, 0,
        0.5, 0.5, 0,
        0.5, -0.5, 0,
    }
)
type cell struct {
    drawable uint32
    alive     bool
    aliveNext bool
    x int
    y int
}
func main() {
    runtime.LockOSThread()
    window := initGlfw()
    defer glfw.Terminate()
    program := initOpenGL()
    cells := makeCells()
    for !window.ShouldClose() {
        t := time.Now()
        for x := range cells {
            for _, c := range cells[x] {
                c.checkState(cells)
            }
        }
        draw(cells, window, program)
        time.Sleep(time.Second/time.Duration(fps) - time.Since(t))
    }
}
func draw(cells [][]*cell, window *glfw.Window, program uint32) {
    gl.Clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT)
    gl.UseProgram(program)
    for x := range cells {
        for _, c := range cells[x] {
            c.draw()
        }
    }
    glfw.PollEvents()
    window.SwapBuffers()
}
func makeCells() [][]*cell {
    rand.Seed(time.Now().UnixNano())
    cells := make([][]*cell, rows, rows)
    for x := 0; x < rows; x++ {
        for y := 0; y < columns; y++ {
            c := newCell(x, y)
            c.alive = rand.Float64() < threshold
            c.aliveNext = c.alive
            cells[x] = append(cells[x], c)
        }
    }
    return cells
}
func newCell(x, y int) *cell {
    points := make([]float32, len(square), len(square))
    copy(points, square)
    for i := 0; i < len(points); i++ {
        var position float32
        var size float32
        switch i % 3 {
        case 0:
            size = 1.0 / float32(columns)
            position = float32(x) * size
        case 1:
            size = 1.0 / float32(rows)
            position = float32(y) * size
        default:
            continue
        }
        if points[i] < 0 {
            points[i] = (position * 2) - 1
        } else {
            points[i] = ((position + size) * 2) - 1
        }
    }
    return &cell{
        drawable: makeVao(points),
        x: x,
        y: y,
    }
}
func (c *cell) draw() {
    if !c.alive {
        return
    }
    gl.BindVertexArray(c.drawable)
    gl.DrawArrays(gl.TRIANGLES, 0, int32(len(square)/3))
}
// checkState 函数决定下一次游戏循环时的 cell 状态
func (c *cell) checkState(cells [][]*cell) {
    c.alive = c.aliveNext
    c.aliveNext = c.alive
    liveCount := c.liveNeighbors(cells)
    if c.alive {
        // 1. 当任何一个存活的 cell 的附近少于 2 个存活的 cell 时，该 cell 将会消亡，就像人口过少所导致的结果一样
        if liveCount < 2 {
            c.aliveNext = false
        }
        // 2. 当任何一个存活的 cell 的附近有 2 至 3 个存活的 cell 时，该 cell 在下一代中仍然存活。
        if liveCount == 2 || liveCount == 3 {
            c.aliveNext = true
        }
        // 3. 当任何一个存活的 cell 的附近多于 3 个存活的 cell 时，该 cell 将会消亡，就像人口过多所导致的结果一样
        if liveCount > 3 {
            c.aliveNext = false
        }
    } else {
        // 4. 任何一个消亡的 cell 附近刚好有 3 个存活的 cell，该 cell 会变为存活的状态，就像重生一样。
        if liveCount == 3 {
            c.aliveNext = true
        }
    }
}
// liveNeighbors 函数返回当前 cell 附近存活的 cell 数
func (c *cell) liveNeighbors(cells [][]*cell) int {
    var liveCount int
    add := func(x, y int) {
        // If we're at an edge, check the other side of the board.
        if x == len(cells) {
            x = 0
        } else if x == -1 {
            x = len(cells) - 1
        }
        if y == len(cells[x]) {
            y = 0
        } else if y == -1 {
            y = len(cells[x]) - 1
        }
        if cells[x][y].alive {
            liveCount++
        }
    }
    add(c.x-1, c.y)   // To the left
    add(c.x+1, c.y)   // To the right
    add(c.x, c.y+1)   // up
    add(c.x, c.y-1)   // down
    add(c.x-1, c.y+1) // top-left
    add(c.x+1, c.y+1) // top-right
    add(c.x-1, c.y-1) // bottom-left
    add(c.x+1, c.y-1) // bottom-right
    return liveCount
}
// initGlfw 初始化 glfw，返回一个可用的 Window
func initGlfw() *glfw.Window {
    if err := glfw.Init(); err != nil {
        panic(err)
    }
    glfw.WindowHint(glfw.Resizable, glfw.False)
    glfw.WindowHint(glfw.ContextVersionMajor, 4)
    glfw.WindowHint(glfw.ContextVersionMinor, 1)
    glfw.WindowHint(glfw.OpenGLProfile, glfw.OpenGLCoreProfile)
    glfw.WindowHint(glfw.OpenGLForwardCompatible, glfw.True)
    window, err := glfw.CreateWindow(width, height, "Conway's Game of Life", nil, nil)
    if err != nil {
        panic(err)
    }
    window.MakeContextCurrent()
    return window
}
// initOpenGL 初始化 OpenGL 并返回一个已经编译好的着色器程序
func initOpenGL() uint32 {
    if err := gl.Init(); err != nil {
        panic(err)
    }
    version := gl.GoStr(gl.GetString(gl.VERSION))
    log.Println("OpenGL version", version)
    vertexShader, err := compileShader(vertexShaderSource, gl.VERTEX_SHADER)
    if err != nil {
        panic(err)
    }
    fragmentShader, err := compileShader(fragmentShaderSource, gl.FRAGMENT_SHADER)
    if err != nil {
        panic(err)
    }
    prog := gl.CreateProgram()
    gl.AttachShader(prog, vertexShader)
    gl.AttachShader(prog, fragmentShader)
    gl.LinkProgram(prog)
    return prog
}
// makeVao 初始化并从提供的点里面返回一个顶点数组
func makeVao(points []float32) uint32 {
    var vbo uint32
    gl.GenBuffers(1, &vbo)
    gl.BindBuffer(gl.ARRAY_BUFFER, vbo)
    gl.BufferData(gl.ARRAY_BUFFER, 4*len(points), gl.Ptr(points), gl.STATIC_DRAW)
    var vao uint32
    gl.GenVertexArrays(1, &vao)
    gl.BindVertexArray(vao)
    gl.EnableVertexAttribArray(0)
    gl.BindBuffer(gl.ARRAY_BUFFER, vbo)
    gl.VertexAttribPointer(0, 3, gl.FLOAT, false, 0, nil)
    return vao
}
func compileShader(source string, shaderType uint32) (uint32, error) {
    shader := gl.CreateShader(shaderType)
    csources, free := gl.Strs(source)
    gl.ShaderSource(shader, 1, csources, nil)
    free()
    gl.CompileShader(shader)
    var status int32
    gl.GetShaderiv(shader, gl.COMPILE_STATUS, &status)
    if status == gl.FALSE {
        var logLength int32
        gl.GetShaderiv(shader, gl.INFO_LOG_LENGTH, &logLength)
        log := strings.Repeat("\x00", int(logLength+1))
        gl.GetShaderInfoLog(shader, logLength, nil, gl.Str(log))
        return 0, fmt.Errorf("failed to compile %v: %v", source, log)
    }
    return shader, nil
}
```

