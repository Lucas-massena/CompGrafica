#include <iostream>
#include <vector>
#include <cmath>
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#define TINYOBJLOADER_IMPLEMENTATION
#include "tiny_obj_loader.h"

//Definido vetores e matrizes
struct Vec3 { float x, y, z; };
struct Mat4 { float matriz[16]; };

//Criando matriz identidade que vai ser usada para translate, rotate etc
Mat4 Identidade() {
    Mat4 resultado{};
    for (int i = 0; i < 16; i++) resultado.matriz[i] = 0;
    resultado.matriz[0] = resultado.matriz[5] = resultado.matriz[10] = resultado.matriz[15] = 1;
    return resultado;
}

//Fazendo multiplicacao de matriz que vai ser usada para as transformacoes do objeto
Mat4 Mul(const Mat4& A, const Mat4& B) {
    Mat4 resultado{};
    for (int r = 0; r < 4; r++)
        for (int c = 0; c < 4; c++)
            resultado.matriz[r * 4 + c] =
            A.matriz[r * 4 + 0] * B.matriz[0 * 4 + c] +
            A.matriz[r * 4 + 1] * B.matriz[1 * 4 + c] +
            A.matriz[r * 4 + 2] * B.matriz[2 * 4 + c] +
            A.matriz[r * 4 + 3] * B.matriz[3 * 4 + c];
    return resultado;
}

//Fazendo a transalacao para centralizar o objeto na origem
Mat4 Translate(float x, float y, float z) {
    Mat4 resultado = Identidade();
    resultado.matriz[12] = x;
    resultado.matriz[13] = y;
    resultado.matriz[14] = z;
    return resultado;
}

//Fazendo o rotate para a movimentacao com o mouse no eixo X
Mat4 RotateX(float a) {
    Mat4 r = Identidade();
    float c = cos(a), s = sin(a);
    r.matriz[5] = c; r.matriz[6] = -s;
    r.matriz[9] = s; r.matriz[10] = c;
    return r;
}

//Fazendo o rotate para a movimentacao com o mouse no eixo y
Mat4 RotateY(float a) {
    Mat4 resultado = Identidade();
    float c = cos(a), s = sin(a);
    resultado.matriz[0] = c; resultado.matriz[2] = s;
    resultado.matriz[8] = -s; resultado.matriz[10] = c;
    return resultado;
}

//Fazendo a matriz que vai ser usada para a projecao perspectiva em 3d
Mat4 Perspective(float fov, float aspect, float n, float f) {
    Mat4 resultado{};
    float t = tan(fov / 2.0f);
    resultado.matriz[0] = 1.0f / (aspect * t);
    resultado.matriz[5] = 1.0f / t;
    resultado.matriz[10] = -(f + n) / (f - n);
    resultado.matriz[11] = -1.0f;
    resultado.matriz[14] = -(2 * f * n) / (f - n);
    return resultado;
}

//Fazendo a matriz da camera
Mat4 LookAt(Vec3 eye, Vec3 center, Vec3 up) {
    Vec3 f{ center.x - eye.x, center.y - eye.y, center.z - eye.z };
    float fl = sqrt(f.x * f.x + f.y * f.y + f.z * f.z);
    f = { f.x / fl, f.y / fl, f.z / fl };

    float ul = sqrt(up.x * up.x + up.y * up.y + up.z * up.z);
    up = { up.x / ul, up.y / ul, up.z / ul };

    Vec3 s{
        f.y * up.z - f.z * up.y,
        f.z * up.x - f.x * up.z,
        f.x * up.y - f.y * up.x
    };
    float sl = sqrt(s.x * s.x + s.y * s.y + s.z * s.z);
    s = { s.x / sl, s.y / sl, s.z / sl };

    Vec3 u{
        s.y * f.z - s.z * f.y,
        s.z * f.x - s.x * f.z,
        s.x * f.y - s.y * f.x
    };

    Mat4 resultado = Identidade();
    resultado.matriz[0] = s.x; resultado.matriz[4] = s.y; resultado.matriz[8] = s.z;
    resultado.matriz[1] = u.x; resultado.matriz[5] = u.y; resultado.matriz[9] = u.z;
    resultado.matriz[2] = -f.x; resultado.matriz[6] = -f.y; resultado.matriz[10] = -f.z;

    return Mul(resultado, Translate(-eye.x, -eye.y, -eye.z));
}


//Shaders
const char* VERTEX_SHADER = R"(
#version 330 core
layout(location=0) in vec3 aPos;
layout(location=1) in vec3 aNormal;

uniform mat4 model, view, proj;

out vec3 N, P;

void main() {
    P = vec3(model * vec4(aPos,1.0));
    N = mat3(model) * aNormal;
    gl_Position = proj * view * vec4(P,1.0);
}
)";

const char* FRAGMENT_SHADER = R"(
#version 330 core
in vec3 N, P;
out vec4 FragColor;
uniform vec3 lightPos;
uniform vec3 viewPos;
uniform sampler2D tex0;
vec2 uvMap(vec3 p) { return p.xz * 0.5; }

void main() {
    vec3 n = normalize(N);
    vec3 l = normalize(lightPos - P);
    vec3 v = normalize(viewPos - P);
    vec3 r = reflect(-l, n);
    float diff = max(dot(n,l), 0.0);
    float spec = pow(max(dot(r,v), 0.0), 32.0);
    vec3 texC = texture(tex0, uvMap(P * 2.0)).rgb;
    vec3 col = 0.12 * texC + diff * texC + spec * vec3(1.0);
    FragColor = vec4(col, 1.0);
}
)";
//Criando shader
GLuint Compile(GLenum type, const char* src) {
    GLuint s = glCreateShader(type);
    glShaderSource(s, 1, &src, nullptr);
    glCompileShader(s);
    return s;
}

//Criando programa
GLuint Programa() {
    GLuint v = Compile(GL_VERTEX_SHADER, VERTEX_SHADER);
    GLuint f = Compile(GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
    GLuint p = glCreateProgram();

    glAttachShader(p, v);
    glAttachShader(p, f);
    glLinkProgram(p);
    glDeleteShader(v);
    glDeleteShader(f);
    return p;
}

//Criando textura
GLuint CreateFurTexture(int size = 512) {
    std::vector<unsigned char> img(size * size * 3);

    for (int y = 0; y < size; y++)
        for (int x = 0; x < size; x++) {

            // Ruído simples para variação suave na cor
            float nx = (float)x / size;
            float ny = (float)y / size;
            float noise = (sin(nx * 20.0f) + sin(ny * 25.0f)) * 0.25f + 0.5f;

            // Cores típicas de coelho marrom claro/bege
            unsigned char r = (unsigned char)(50+ noise * 50);  // tons entre 170–210
            unsigned char g = (unsigned char)(80 + noise * 60);  // tons entre 140–175
            unsigned char b = (unsigned char)(180 + noise * 50);  // tons entre 110–140

            int i = (y * size + x) * 3;
            img[i + 0] = r;
            img[i + 1] = g;
            img[i + 2] = b;
        }

    GLuint t;
    glGenTextures(1, &t);
    glBindTexture(GL_TEXTURE_2D, t);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, size, size, 0,
        GL_RGB, GL_UNSIGNED_BYTE, img.data());
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glGenerateMipmap(GL_TEXTURE_2D);

    return t;
}

//Configurando movimentos de camera com o mouse
bool mouseDown = false;
double lastX, lastY;
float yaw = 0, pitch = 0;
const float SENS = 0.005f;

//Detectando  clique no botao esquerdo
void MouseButton(GLFWwindow* w, int b, int act, int mods) {
    if (b == GLFW_MOUSE_BUTTON_LEFT)
        mouseDown = (act == GLFW_PRESS);
    glfwGetCursorPos(w, &lastX, &lastY);
}
//Fazendo a movimentacao da camera com o mouse apos o clique
void MouseMove(GLFWwindow* w, double x, double y) {
    if (!mouseDown) return;
    yaw += (x - lastX) * SENS;
    pitch += (y - lastY) * SENS;
    lastX = x; lastY = y;

    pitch = std::max(std::min(pitch, 1.4f), -1.4f);
}

//Carregando objeto
struct Vertex { float px, py, pz, nx, ny, nz; };
bool LoadOBJ(const char* path,
    std::vector<Vertex>& verts,
    std::vector<unsigned>& idx)
{
    tinyobj::attrib_t attrib;
    std::vector<tinyobj::shape_t> shapes;
    std::vector<tinyobj::material_t> mats;
    std::string warn, err;
    bool ok = tinyobj::LoadObj(&attrib, &shapes, &mats, &warn, &err, path);
    if (!ok || shapes.empty()) return false;
    for (auto& s : shapes)
        for (auto& i : s.mesh.indices) {
            Vertex v{};
            int vi = i.vertex_index * 3;
        int ni = i.normal_index * 3;
            v.px = attrib.vertices[vi];
            v.py = attrib.vertices[vi + 1];
            v.pz = attrib.vertices[vi + 2];

            if (i.normal_index >= 0) {
                v.nx = attrib.normals[ni];
                v.ny = attrib.normals[ni + 1];
                v.nz = attrib.normals[ni + 2];
            }
            else v.nx = 0, v.ny = 1, v.nz = 0;
            verts.push_back(v);
            idx.push_back((unsigned)idx.size());
        }
    return true;
}

//Funcao main
int main() {
    //Iniciando a janela
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    GLFWwindow* w = glfwCreateWindow(1200, 1000, "Coelho de Stanford", nullptr, nullptr);
    glfwMakeContextCurrent(w);
    gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);
    glEnable(GL_DEPTH_TEST);
    glfwSetMouseButtonCallback(w, MouseButton);
    glfwSetCursorPosCallback(w, MouseMove);

    //Carregando objeto
    std::vector<Vertex> verts;
    std::vector<unsigned> idx;
    LoadOBJ("models/bunny.obj", verts, idx);

    //Esfera delimitadora
    Vec3 minv{ 1e9,1e9,1e9 }, maxv{ -1e9,-1e9,-1e9 };
    for (auto& v : verts) {
        minv.x = std::min(minv.x, v.px);
        minv.y = std::min(minv.y, v.py);
        minv.z = std::min(minv.z, v.pz);
        maxv.x = std::max(maxv.x, v.px);
        maxv.y = std::max(maxv.y, v.py);
        maxv.z = std::max(maxv.z, v.pz);
    }
    Vec3 center{
        (minv.x + maxv.x) / 2,
        (minv.y + maxv.y) / 2,
        (minv.z + maxv.z) / 2
    };
    float radius = sqrt(
        pow(maxv.x - center.x, 2) +
        pow(maxv.y - center.y, 2) +
        pow(maxv.z - center.z, 2));

    //Criando VAO, VBO e EBO
    GLuint VAO, VBO, EBO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);
    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, verts.size() * sizeof(Vertex), verts.data(), GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, idx.size() * sizeof(unsigned), idx.data(), GL_STATIC_DRAW);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)0);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(3 * sizeof(float)));
    glEnableVertexAttribArray(1);
    glBindVertexArray(0);
    GLuint shader = Programa();
    GLuint tex = CreateFurTexture();
    Vec3 camPos{ 0,0, radius * 3 };

    while (!glfwWindowShouldClose(w)) {
        //Eventos + limpando a tela
        glfwPollEvents();
        glClearColor(0.12f, 0.12f, 0.14f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

        //Matrizes
        Mat4 model = Translate(-center.x, -center.y, -center.z);
        model = Mul(RotateX(pitch), model);
        model = Mul(RotateY(yaw), model);
        Mat4 view = LookAt(camPos, { 0,0,0 }, { 0,1,0 });
        Mat4 proj = Perspective(3.1415f / 4, 800.f / 600.f, 0.01f, radius * 50);

        //Uniforms
        glUseProgram(shader);
        glUniformMatrix4fv(glGetUniformLocation(shader, "model"), 1, GL_FALSE, model.matriz);
        glUniformMatrix4fv(glGetUniformLocation(shader, "view"), 1, GL_FALSE, view.matriz);
        glUniformMatrix4fv(glGetUniformLocation(shader, "proj"), 1, GL_FALSE, proj.matriz);
        glUniform3f(glGetUniformLocation(shader, "lightPos"), radius * 2, radius * 2, radius * 2);
        glUniform3f(glGetUniformLocation(shader, "viewPos"), camPos.x, camPos.y, camPos.z);
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, tex);
        glUniform1i(glGetUniformLocation(shader, "tex0"), 0);

        //Desenhando
        glBindVertexArray(VAO);
        glDrawElements(GL_TRIANGLES, idx.size(), GL_UNSIGNED_INT, 0);

        glfwSwapBuffers(w);
    }

    return 0;
}
