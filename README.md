# Trabalho-Final-CG
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

 let scene, camera, renderer, controls;
    let dividingPlane;
    let transparentRT, reflectedRT;
    let clipPlaneTransparent, clipPlaneReflected;

 // Shader customizado para os cubos com clipping planes (Requisito 5: Clipping Plane)
    var CustomCubeShader = {
      uniforms: {
        cubeTexture: { value: null } // Textura padrão para os cubos (Requisito 4)
      },
      vertexShader: `
        // Vertex Shader do cubo
        // Inclui parâmetros para clipping e calcula as coordenadas de textura
        #include <clipping_planes_pars_vertex>
        out vec2 vUv;
        void main() {
          vUv = uv;
          vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
          gl_Position = projectionMatrix * mvPosition;
          #if NUM_CLIPPING_PLANES > 0
            vClipPosition = -mvPosition.xyz;
          #endif
        }
      `,
      fragmentShader: `
        // Fragment Shader do cubo
        // Aplica a textura padrão e utiliza o clipping plane
        precision mediump float;
        in vec2 vUv;
        #include <clipping_planes_pars_fragment>
        uniform sampler2D cubeTexture;
        out vec4 outColor;
        void main() {
          #include <clipping_planes_fragment>
          outColor = texture(cubeTexture, vUv);
        }
      `
    };
  // =======================================================================
    // REQUISITO 1 (continuação) e Variação: 
    // - Shader customizado para efeito Fresnel que simula reflexão e transparência.
    // - Calcula a reflexão baseada no ângulo de visão.
    // =======================================================================
    var CustomFresnelShader = {
      uniforms: {
        "mFresnelBias": { value: 0.1 },
        "mFresnelPower": { value: 2.0 },
        "mFresnelScale": { value: 1.0 },
        "transparency": { value: 0.8 }, // REQUISITO 3: Transparência controlada por uniform
        "transparentTexture": { value: null }, // Render target para a cena transparente
        "reflectedTexture": { value: null }    // Render target para a cena refletida
      },
      vertexShader: `
          // Vertex Shader para efeito Fresnel
          // Calcula a reflexão com base no ângulo de visão (interação dos materiais translúcidos com o ambiente)
          uniform float mFresnelBias;
          uniform float mFresnelPower;
          uniform float mFresnelScale;
          out vec2 vUv;
          out float vReflectionFactor;
          out vec3 vWorldPosition;
          void main() {
            vUv = uv;
            vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
            vec4 worldPosition = modelMatrix * vec4(position, 1.0);
            vWorldPosition = worldPosition.xyz;
            vec3 I = worldPosition.xyz - cameraPosition;
            vec3 worldNormal = normalize(mat3(modelMatrix) * normal);
            // Cálculo do fator de reflexão baseado no ângulo de visão
            vReflectionFactor = mFresnelBias + mFresnelScale * pow(1.0 + dot(normalize(I), worldNormal), mFresnelPower);
            gl_Position = projectionMatrix * mvPosition;
          }
        `,
      fragmentShader: `
          // Fragment Shader para efeito Fresnel
          // Combina os efeitos de reflexo e transparência usando render targets
          uniform sampler2D transparentTexture;
          uniform sampler2D reflectedTexture;
          uniform float transparency;
          in vec2 vUv;
          in float vReflectionFactor;
          in vec3 vWorldPosition;
          out vec4 fragColor;
          void main() {
            // Clipping plane: descarta fragmentos abaixo de uma determinada posição (Requisito 5)
            if (vWorldPosition.y < -2.0) discard;
            // Obtém cores das cenas renderizadas: transparente e refletida (Requisito 2)
            vec4 transColor = texture(transparentTexture, rippleUv);
            vec4 reflColor  = texture(reflectedTexture, rippleUv);
            // Mistura os dois efeitos com base no fator de reflexão
            float blendFactor = smoothstep(-0.2, 0.2, vWorldPosition.x);
            float mixFactor = mix(vReflectionFactor, 1.0 - vReflectionFactor, blendFactor);
            vec4 finalColor = mix(transColor, reflColor, clamp(mixFactor, 0.0, 1.0));
            finalColor.a *= transparency; // Aplica a transparência controlada pelo uniform
            fragColor = finalColor;
          }
        `
    };



function init() {
    // Inicialização do renderer com WebGL (WebGL/GLSL conforme o enunciado)
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.localClippingEnabled = true; // Ativa clipping planes (Requisito 5)
      document.body.appendChild(renderer.domElement);
    
      // Criação da cena e câmera
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
      camera.position.set(0, 2, 10);
      controls = new OrbitControls(camera, renderer.domElement);
    
      // Definição do ambiente com uma textura ambiente (textura padrão para a cena)
      const ambientTextureLoader = new THREE.CubeTextureLoader();
      const ambientTexture = ambientTextureLoader.load([
        'paisagem.png', 'paisagem.png', 'paisagem.png',
        'paisagem.png', 'paisagem.png', 'paisagem.png'
      ]);
      scene.background = ambientTexture;

       // =======================================================================
      // CUBOS: Criação de dois cubos com CustomCubeShader (textura padrão e clipping)
      // =======================================================================
      const cubeGeometry = new THREE.BoxGeometry(1, 1, 1);
      const cubeMaterial = new THREE.ShaderMaterial({
        uniforms: THREE.UniformsUtils.clone(CustomCubeShader.uniforms),
        vertexShader: CustomCubeShader.vertexShader,
        fragmentShader: CustomCubeShader.fragmentShader,
        glslVersion: THREE.GLSL3,
        clipping: true
      });
   
// =======================================================================
      // REQUISITO 2: Reflexo e Transparência Dinâmicos
      // - Render targets para cenas transparente e refletida, simulando o efeito sem cube map.
      // =======================================================================
      transparentRT = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
        minFilter: THREE.LinearFilter,
        magFilter: THREE.LinearFilter,
        format: THREE.RGBAFormat
      });
      reflectedRT = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
        minFilter: THREE.LinearFilter,
        magFilter: THREE.LinearFilter,
        format: THREE.RGBAFormat
      });

// Clipping planes para separar as cenas de acordo com a posição (Requisito 5)
      clipPlaneTransparent = new THREE.Plane(new THREE.Vector3(1, 0, 0), 0.2);
      clipPlaneReflected = new THREE.Plane(new THREE.Vector3(1, 0, 0), -0.2);

  // =======================================================================
      // PLANO DIVISOR com efeito Fresnel (shader customizado)
      // - Combina as duas cenas (transparente e refletida) e aplica o efeito de bolha de sabão.
      // =======================================================================
      const planeGeometry = new THREE.PlaneGeometry(6, 6);
      const fresnelUniforms = THREE.UniformsUtils.clone(CustomFresnelShader.uniforms);
      // Passa os render targets como texturas para o shader
      fresnelUniforms.transparentTexture.value = transparentRT.texture;
      fresnelUniforms.reflectedTexture.value = reflectedRT.texture;
      const fresnelMaterial = new THREE.ShaderMaterial({
        uniforms: fresnelUniforms,
        vertexShader: CustomFresnelShader.vertexShader,
        fragmentShader: CustomFresnelShader.fragmentShader,
        transparent: true,
        glslVersion: THREE.GLSL3
      });
      dividingPlane = new THREE.Mesh(planeGeometry, fresnelMaterial);
      dividingPlane.position.set(0, 0, 0);
      dividingPlane.rotation.y = Math.PI / 2;
      dividingPlane.renderOrder = 1; // Garante que o plano seja renderizado após os outros objetos
      scene.add(dividingPlane);

      window.addEventListener('resize', onWindowResize, false);
    }


    function onWindowResize() {
          // Atualiza câmera e render targets quando a janela é redimensionada
          camera.aspect = window.innerWidth / window.innerHeight;
          camera.updateProjectionMatrix();
          renderer.setSize(window.innerWidth, window.innerHeight);
          transparentRT.setSize(window.innerWidth, window.innerHeight);
          reflectedRT.setSize(window.innerWidth, window.innerHeight);
        }
    
 // =======================================================================
    // Loop de animação: renderiza a cena em tempo real
    // - Executa três etapas de renderização (Requisito 2)
    //   1. Renderiza a cena para a textura transparente com clipping plane.
    //   2. Renderiza a cena para a textura refletida com clipping plane.
    //   3. Renderiza a cena final combinando ambas as texturas no plano com efeito Fresnel.
    // =======================================================================
    function animate() {
      requestAnimationFrame(animate);
      controls.update();

      // Primeiro passe: renderiza cena transparente (clipping plane para separar)
      dividingPlane.visible = false;
      renderer.clippingPlanes = [clipPlaneTransparent];
      renderer.setRenderTarget(transparentRT);
      renderer.clear();
      renderer.render(scene, camera);

      // Segundo passe: renderiza cena refletida (clipping plane inverso)
      renderer.clippingPlanes = [clipPlaneReflected];
      renderer.setRenderTarget(reflectedRT);
      renderer.clear();
      renderer.render(scene, camera);

      // Terceiro passe: renderiza cena final com o plano dividindo as duas cenas
      renderer.setRenderTarget(null);
      renderer.clippingPlanes = [];
      dividingPlane.visible = true;
      // Atualiza os uniforms do shader Fresnel com as texturas atualizadas
      dividingPlane.material.uniforms.transparentTexture.value = transparentRT.texture;
      dividingPlane.material.uniforms.reflectedTexture.value = reflectedRT.texture;
      renderer.render(scene, camera);
    }
  </script>

















      
