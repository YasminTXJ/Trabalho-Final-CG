# Trabalho-Final-CG
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

let scene, camera, renderer, renderTargetRefraction, renderTargetReflection, planeMaterial, planeMesh, controls;

function init() {
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.25, 100);
    camera.position.set(-2, 0.6, 3);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    controls = new OrbitControls(camera, renderer.domElement);
    window.addEventListener('resize', onWindowResize);

    const light = new THREE.PointLight(0xffffff, 1, 100);
    light.position.set(2, 5, 3);
    scene.add(light);

    const cube = new THREE.Mesh(
        new THREE.BoxGeometry(1, 1, 1),
        new THREE.MeshStandardMaterial({ color: 0xff0000 })
    );
    cube.position.set(0, 0, 2);
    scene.add(cube);

    const sphere = new THREE.Mesh(
        new THREE.SphereGeometry(1, 32, 32),
        new THREE.MeshStandardMaterial({ color: 0x0000ff })
    );
    sphere.position.set(0, 0, -2);
    scene.add(sphere);

    renderTargetRefraction = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight);
    renderTargetReflection = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight);

    const planeGeometry = new THREE.PlaneGeometry(10, 10);
    planeMaterial = new THREE.ShaderMaterial({
        uniforms: {
            refractionTexture: { value: renderTargetRefraction.texture },
            reflectionTexture: { value: renderTargetReflection.texture },
            mixFactor: { value: 0.5 } // Controla a mistura refração/reflexão
        },
        vertexShader: `
            varying vec2 vUv;
            void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D refractionTexture;
            uniform sampler2D reflectionTexture;
            uniform float mixFactor;
            varying vec2 vUv;
            void main() {
                vec4 refractColor = texture2D(refractionTexture, vUv);
                vec4 reflectColor = texture2D(reflectionTexture, vUv);
                gl_FragColor = mix(refractColor, reflectColor, mixFactor);
            }
        `,
        transparent: true
    });

    planeMesh = new THREE.Mesh(planeGeometry, planeMaterial);
    planeMesh.position.set(0, 1, 0);
    scene.add(planeMesh);

    animate();
}

function renderSceneToTarget(target, clipPlane) {
    renderer.setRenderTarget(target);
    renderer.clippingPlanes = clipPlane ? [clipPlane] : [];
    renderer.render(scene, camera);
    renderer.setRenderTarget(null);
}

function animate() {
    requestAnimationFrame(animate);

    const clipPlane = new THREE.Plane(new THREE.Vector3(0, 0, -1), 0.5);
    renderSceneToTarget(renderTargetRefraction, clipPlane);
    renderSceneToTarget(renderTargetReflection);

    renderer.render(scene, camera);
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}

init();
