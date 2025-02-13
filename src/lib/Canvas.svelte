<script>
	import { onMount } from 'svelte';
	import * as shaders from './shaders';

	export let SIM_RESOLUTION = 128;
	export let DYE_RESOLUTION = 1024;
	export let CAPTURE_RESOLUTION = 512;
	export let DENSITY_DISSIPATION = 1;
	export let VELOCITY_DISSIPATION = 0.2;
	export let PRESSURE = 0.8;
	export let PRESSURE_ITERATIONS = 20;
	export let CURL = 30;
	export let SPLAT_RADIUS = 0.25;
	export let SPLAT_FORCE = 6000;
	export let SHADING = true;
	export let COLORFUL = true;
	export let COLOR_UPDATE_SPEED = 10;
	export let PAUSED = false;
	export let BACK_COLOR = { r: 0, g: 0, b: 0 };
	export let TRANSPARENT = false;
	export let BLOOM = true;
	export let BLOOM_ITERATIONS = 8;
	export let BLOOM_RESOLUTION = 256;
	export let BLOOM_INTENSITY = 0.8;
	export let BLOOM_THRESHOLD = 0.6;
	export let BLOOM_SOFT_KNEE = 0.7;
	export let SUNRAYS = true;
	export let SUNRAYS_RESOLUTION = 196;
	export let SUNRAYS_WEIGHT = 1.0;

	// a bit hackish, these actions are used by the knobby controls component
	export let actions;
	Object.assign(actions, { randomSplats, captureScreenshot });

	// should work similarly to dat.gui's onFinishChange hook
	$: SIM_RESOLUTION,
		DYE_RESOLUTION,
		BLOOM_ITERATIONS,
		BLOOM_RESOLUTION,
		SUNRAYS_RESOLUTION,
		gl && initFramebuffers();
	$: SHADING, BLOOM, SUNRAYS, displayMaterial && updateKeywords();

	function getSupportedFormat(gl, internalFormat, format, type) {
		const texture = gl.createTexture();
		gl.bindTexture(gl.TEXTURE_2D, texture);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
		gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, 4, 4, 0, format, type, null);

		const fbo = gl.createFramebuffer();
		gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
		gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);

		const status = gl.checkFramebufferStatus(gl.FRAMEBUFFER);
		const supportRenderTextureFormat = status == gl.FRAMEBUFFER_COMPLETE;

		if (!supportRenderTextureFormat) {
			switch (internalFormat) {
				case gl.R16F:
					return getSupportedFormat(gl, gl.RG16F, gl.RG, type);
				case gl.RG16F:
					return getSupportedFormat(gl, gl.RGBA16F, gl.RGBA, type);
				default:
					return null;
			}
		}

		return {
			internalFormat,
			format
		};
	}

	const isMobile = () => /Mobi|Android/i.test(navigator.userAgent);

	const createPointer = () => ({
		id: -1,
		texcoordX: 0,
		texcoordY: 0,
		prevTexcoordX: 0,
		prevTexcoordY: 0,
		deltaX: 0,
		deltaY: 0,
		down: false,
		moved: false,
		color: [30, 0, 300]
	});

	function updatePointerDownData(pointer, id, posX, posY) {
		pointer.id = id;
		pointer.down = true;
		pointer.moved = false;
		pointer.texcoordX = posX / canvas.width;
		pointer.texcoordY = 1.0 - posY / canvas.height;
		pointer.prevTexcoordX = pointer.texcoordX;
		pointer.prevTexcoordY = pointer.texcoordY;
		pointer.deltaX = 0;
		pointer.deltaY = 0;
		pointer.color = generateColor();
	}

	function updatePointerMoveData(pointer, posX, posY) {
		pointer.prevTexcoordX = pointer.texcoordX;
		pointer.prevTexcoordY = pointer.texcoordY;
		pointer.texcoordX = posX / canvas.width;
		pointer.texcoordY = 1.0 - posY / canvas.height;
		const aspectRatio = canvas.width / canvas.height;
		pointer.deltaX = correctDeltaX(pointer.texcoordX - pointer.prevTexcoordX, aspectRatio);
		pointer.deltaY = correctDeltaY(pointer.texcoordY - pointer.prevTexcoordY, aspectRatio);
		pointer.moved = Math.abs(pointer.deltaX) > 0 || Math.abs(pointer.deltaY) > 0;
	}

	const correctDeltaX = (delta, aspectRatio) => (aspectRatio < 1 ? delta * aspectRatio : delta);

	const correctDeltaY = (delta, aspectRatio) => (aspectRatio > 1 ? delta / aspectRatio : delta);

	function generateColor() {
		const c = HSVtoRGB(Math.random(), 1.0, 1.0);
		c.r *= 0.15;
		c.g *= 0.15;
		c.b *= 0.15;
		return c;
	}

	function HSVtoRGB(h, s, v) {
		let r, g, b, i, f, p, q, t;
		i = Math.floor(h * 6);
		f = h * 6 - i;
		p = v * (1 - s);
		q = v * (1 - f * s);
		t = v * (1 - (1 - f) * s);

		switch (i % 6) {
			case 0:
				(r = v), (g = t), (b = p);
				break;
			case 1:
				(r = q), (g = v), (b = p);
				break;
			case 2:
				(r = p), (g = v), (b = t);
				break;
			case 3:
				(r = p), (g = q), (b = v);
				break;
			case 4:
				(r = t), (g = p), (b = v);
				break;
			case 5:
				(r = v), (g = p), (b = q);
				break;
		}

		return {
			r,
			g,
			b
		};
	}

	const normalizeColor = (input) => ({
		r: input.r / 255,
		g: input.g / 255,
		b: input.b / 255
	});

	function wrap(value, min, max) {
		const range = max - min;
		if (range == 0) return min;
		return ((value - min) % range) + min;
	}

	const getTextureScale = (texture, width, height) => ({
		x: width / texture.width,
		y: height / texture.height
	});

	const scaleByPixelRatio = (input) => Math.floor(input * (window.devicePixelRatio || 1));

	function hashCode(s) {
		if (s.length == 0) return 0;
		let hash = 0;
		for (let i = 0; i < s.length; i++) {
			hash = (hash << 5) - hash + s.charCodeAt(i);
			hash |= 0; // Convert to 32bit integer
		}
		return hash;
	}

	function captureScreenshot() {
		const res = getResolution(CAPTURE_RESOLUTION);
		const target = createFBO(
			res.width,
			res.height,
			ext.formatRGBA.internalFormat,
			ext.formatRGBA.format,
			ext.halfFloatTexType,
			gl.NEAREST
		);
		render(target);

		let texture = framebufferToTexture(target);
		texture = normalizeTexture(texture, target.width, target.height);

		const captureCanvas = textureToCanvas(texture, target.width, target.height);
		const datauri = captureCanvas.toDataURL();
		downloadURI('fluid.png', datauri);
		URL.revokeObjectURL(datauri);
	}

	function framebufferToTexture(target) {
		gl.bindFramebuffer(gl.FRAMEBUFFER, target.fbo);
		const length = target.width * target.height * 4;
		const texture = new Float32Array(length);
		gl.readPixels(0, 0, target.width, target.height, gl.RGBA, gl.FLOAT, texture);
		return texture;
	}

	function normalizeTexture(texture, width, height) {
		const result = new Uint8Array(texture.length);
		let id = 0;
		for (let i = height - 1; i >= 0; i--) {
			for (let j = 0; j < width; j++) {
				const nid = i * width * 4 + j * 4;
				result[nid + 0] = clamp01(texture[id + 0]) * 255;
				result[nid + 1] = clamp01(texture[id + 1]) * 255;
				result[nid + 2] = clamp01(texture[id + 2]) * 255;
				result[nid + 3] = clamp01(texture[id + 3]) * 255;
				id += 4;
			}
		}
		return result;
	}

	const clamp01 = (input) => Math.min(Math.max(input, 0), 1);

	function textureToCanvas(texture, width, height) {
		const captureCanvas = document.createElement('canvas');
		const ctx = captureCanvas.getContext('2d');
		captureCanvas.width = width;
		captureCanvas.height = height;

		const imageData = ctx.createImageData(width, height);
		imageData.data.set(texture);
		ctx.putImageData(imageData, 0, 0);

		return captureCanvas;
	}

	function downloadURI(filename, uri) {
		const link = document.createElement('a');
		link.download = filename;
		link.href = uri;
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
	}

	function createMaterial(vertexShader, fragmentShaderSource) {
		const programs = [];
		let activeProgram = null;
		let uniforms = [];
		return {
			programs,
			get activeProgram() {
				return activeProgram;
			},
			get uniforms() {
				return uniforms;
			},
			setKeywords(keywords) {
				let hash = 0;
				for (let i = 0; i < keywords.length; i++) hash += hashCode(keywords[i]);

				let program = programs[hash];
				if (program == null) {
					const fragmentShader = compileShader(gl.FRAGMENT_SHADER, fragmentShaderSource, keywords);
					program = buildProgram(vertexShader, fragmentShader);
					programs[hash] = program;
				}

				if (program == activeProgram) return;

				uniforms = getUniforms(program);
				activeProgram = program;
			},
			bind() {
				gl.useProgram(activeProgram);
			}
		};
	}

	function createProgram(vertexShader, fragmentShader) {
		const program = buildProgram(vertexShader, fragmentShader);
		const uniforms = getUniforms(program);
		return {
			program,
			uniforms,
			bind() {
				gl.useProgram(program);
			}
		};
	}

	function buildProgram(vertexShader, fragmentShader) {
		const program = gl.createProgram();
		gl.attachShader(program, vertexShader);
		gl.attachShader(program, fragmentShader);
		gl.linkProgram(program);

		if (!gl.getProgramParameter(program, gl.LINK_STATUS))
			console.trace(gl.getProgramInfoLog(program));

		return program;
	}

	function getUniforms(program) {
		const uniforms = [];
		const uniformCount = gl.getProgramParameter(program, gl.ACTIVE_UNIFORMS);
		for (let i = 0; i < uniformCount; i++) {
			const uniformName = gl.getActiveUniform(program, i).name;
			uniforms[uniformName] = gl.getUniformLocation(program, uniformName);
		}
		return uniforms;
	}

	function compileShader(type, source, keywords) {
		source = addKeywords(source, keywords);

		const shader = gl.createShader(type);
		gl.shaderSource(shader, source);
		gl.compileShader(shader);

		if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS))
			console.trace(gl.getShaderInfoLog(shader));

		return shader;
	}

	function addKeywords(source, keywords) {
		if (keywords == null) return source;
		let keywordsString = '';
		keywords.forEach((keyword) => {
			keywordsString += '#define ' + keyword + '\n';
		});
		return keywordsString + source;
	}

	function createBlit(gl) {
		gl.bindBuffer(gl.ARRAY_BUFFER, gl.createBuffer());
		gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1, -1, -1, 1, 1, 1, 1, -1]), gl.STATIC_DRAW);
		gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, gl.createBuffer());
		gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array([0, 1, 2, 0, 2, 3]), gl.STATIC_DRAW);
		gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0);
		gl.enableVertexAttribArray(0);

		return (target, clear = false) => {
			if (target == null) {
				gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
				gl.bindFramebuffer(gl.FRAMEBUFFER, null);
			} else {
				gl.viewport(0, 0, target.width, target.height);
				gl.bindFramebuffer(gl.FRAMEBUFFER, target.fbo);
			}
			if (clear) {
				gl.clearColor(0.0, 0.0, 0.0, 1.0);
				gl.clear(gl.COLOR_BUFFER_BIT);
			}
			// CHECK_FRAMEBUFFER_STATUS();
			gl.drawElements(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0);
		};
	}

	function CHECK_FRAMEBUFFER_STATUS() {
		const status = gl.checkFramebufferStatus(gl.FRAMEBUFFER);
		if (status != gl.FRAMEBUFFER_COMPLETE) console.trace('Framebuffer error: ' + status);
	}

	function initFramebuffers() {
		const simRes = getResolution(SIM_RESOLUTION);
		const dyeRes = getResolution(DYE_RESOLUTION);

		const texType = ext.halfFloatTexType;
		const rgba = ext.formatRGBA;
		const rg = ext.formatRG;
		const r = ext.formatR;
		const filtering = ext.supportLinearFiltering ? gl.LINEAR : gl.NEAREST;

		gl.disable(gl.BLEND);

		if (dye == null)
			dye = createDoubleFBO(
				dyeRes.width,
				dyeRes.height,
				rgba.internalFormat,
				rgba.format,
				texType,
				filtering
			);
		else
			dye = resizeDoubleFBO(
				dye,
				dyeRes.width,
				dyeRes.height,
				rgba.internalFormat,
				rgba.format,
				texType,
				filtering
			);

		if (velocity == null)
			velocity = createDoubleFBO(
				simRes.width,
				simRes.height,
				rg.internalFormat,
				rg.format,
				texType,
				filtering
			);
		else
			velocity = resizeDoubleFBO(
				velocity,
				simRes.width,
				simRes.height,
				rg.internalFormat,
				rg.format,
				texType,
				filtering
			);

		divergence = createFBO(
			simRes.width,
			simRes.height,
			r.internalFormat,
			r.format,
			texType,
			gl.NEAREST
		);
		curl = createFBO(simRes.width, simRes.height, r.internalFormat, r.format, texType, gl.NEAREST);
		pressure = createDoubleFBO(
			simRes.width,
			simRes.height,
			r.internalFormat,
			r.format,
			texType,
			gl.NEAREST
		);

		initBloomFramebuffers();
		initSunraysFramebuffers();
	}

	function initBloomFramebuffers() {
		const res = getResolution(BLOOM_RESOLUTION);

		const texType = ext.halfFloatTexType;
		const rgba = ext.formatRGBA;
		const filtering = ext.supportLinearFiltering ? gl.LINEAR : gl.NEAREST;

		bloom = createFBO(res.width, res.height, rgba.internalFormat, rgba.format, texType, filtering);

		bloomFramebuffers.length = 0;
		for (let i = 0; i < BLOOM_ITERATIONS; i++) {
			const width = res.width >> (i + 1);
			const height = res.height >> (i + 1);

			if (width < 2 || height < 2) break;

			const fbo = createFBO(width, height, rgba.internalFormat, rgba.format, texType, filtering);
			bloomFramebuffers.push(fbo);
		}
	}

	function initSunraysFramebuffers() {
		const res = getResolution(SUNRAYS_RESOLUTION);

		const texType = ext.halfFloatTexType;
		const r = ext.formatR;
		const filtering = ext.supportLinearFiltering ? gl.LINEAR : gl.NEAREST;

		sunrays = createFBO(res.width, res.height, r.internalFormat, r.format, texType, filtering);
		sunraysTemp = createFBO(res.width, res.height, r.internalFormat, r.format, texType, filtering);
	}

	function createFBO(w, h, internalFormat, format, type, param) {
		gl.activeTexture(gl.TEXTURE0);
		const texture = gl.createTexture();
		gl.bindTexture(gl.TEXTURE_2D, texture);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, param);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, param);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
		gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, w, h, 0, format, type, null);

		const fbo = gl.createFramebuffer();
		gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
		gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);
		gl.viewport(0, 0, w, h);
		gl.clear(gl.COLOR_BUFFER_BIT);

		const texelSizeX = 1.0 / w;
		const texelSizeY = 1.0 / h;

		return {
			texture,
			fbo,
			width: w,
			height: h,
			texelSizeX,
			texelSizeY,
			attach(id) {
				gl.activeTexture(gl.TEXTURE0 + id);
				gl.bindTexture(gl.TEXTURE_2D, texture);
				return id;
			}
		};
	}

	function createDoubleFBO(w, h, internalFormat, format, type, param) {
		let fbo1 = createFBO(w, h, internalFormat, format, type, param);
		let fbo2 = createFBO(w, h, internalFormat, format, type, param);

		return {
			width: w,
			height: h,
			texelSizeX: fbo1.texelSizeX,
			texelSizeY: fbo1.texelSizeY,
			get read() {
				return fbo1;
			},
			set read(value) {
				fbo1 = value;
			},
			get write() {
				return fbo2;
			},
			set write(value) {
				fbo2 = value;
			},
			swap() {
				const temp = fbo1;
				fbo1 = fbo2;
				fbo2 = temp;
			}
		};
	}

	function resizeFBO(target, w, h, internalFormat, format, type, param) {
		const newFBO = createFBO(w, h, internalFormat, format, type, param);
		copyProgram.bind();
		gl.uniform1i(copyProgram.uniforms.uTexture, target.attach(0));
		blit(newFBO);
		return newFBO;
	}

	function resizeDoubleFBO(target, w, h, internalFormat, format, type, param) {
		if (target.width == w && target.height == h) return target;
		target.read = resizeFBO(target.read, w, h, internalFormat, format, type, param);
		target.write = createFBO(w, h, internalFormat, format, type, param);
		target.width = w;
		target.height = h;
		target.texelSizeX = 1.0 / w;
		target.texelSizeY = 1.0 / h;
		return target;
	}

	function createTextureAsync(url) {
		const texture = gl.createTexture();
		gl.bindTexture(gl.TEXTURE_2D, texture);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
		gl.texImage2D(
			gl.TEXTURE_2D,
			0,
			gl.RGB,
			1,
			1,
			0,
			gl.RGB,
			gl.UNSIGNED_BYTE,
			new Uint8Array([255, 255, 255])
		);

		const obj = {
			texture,
			width: 1,
			height: 1,
			attach(id) {
				gl.activeTexture(gl.TEXTURE0 + id);
				gl.bindTexture(gl.TEXTURE_2D, texture);
				return id;
			}
		};

		const image = new Image();
		image.onload = () => {
			obj.width = image.width;
			obj.height = image.height;
			gl.bindTexture(gl.TEXTURE_2D, texture);
			gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image);
		};
		image.src = url;

		return obj;
	}

	function updateKeywords() {
		const displayKeywords = [];
		if (SHADING) displayKeywords.push('SHADING');
		if (BLOOM) displayKeywords.push('BLOOM');
		if (SUNRAYS) displayKeywords.push('SUNRAYS');
		displayMaterial.setKeywords(displayKeywords);
	}

	function update() {
		const dt = calcDeltaTime();
		if (resizeCanvas()) initFramebuffers();
		updateColors(dt);
		applyInputs();
		if (!PAUSED) step(dt);
		render(null);
		requestAnimationFrame(update);
	}

	function calcDeltaTime() {
		const now = Date.now();
		let dt = (now - lastUpdateTime) / 1000;
		dt = Math.min(dt, 0.016666);
		lastUpdateTime = now;
		return dt;
	}

	function resizeCanvas() {
		const width = scaleByPixelRatio(canvas.clientWidth);
		const height = scaleByPixelRatio(canvas.clientHeight);
		if (canvas.width != width || canvas.height != height) {
			canvas.width = width;
			canvas.height = height;
			return true;
		}
		return false;
	}

	function updateColors(dt) {
		if (!COLORFUL) return;

		colorUpdateTimer += dt * COLOR_UPDATE_SPEED;
		if (colorUpdateTimer >= 1) {
			colorUpdateTimer = wrap(colorUpdateTimer, 0, 1);
			pointers.forEach((p) => {
				p.color = generateColor();
			});
		}
	}

	function applyInputs() {
		if (splatStack.length > 0) multipleSplats(splatStack.pop());

		pointers.forEach((p) => {
			if (p.moved) {
				p.moved = false;
				splatPointer(p);
			}
		});
	}

	function step(dt) {
		gl.disable(gl.BLEND);

		curlProgram.bind();
		gl.uniform2f(curlProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
		gl.uniform1i(curlProgram.uniforms.uVelocity, velocity.read.attach(0));
		blit(curl);

		vorticityProgram.bind();
		gl.uniform2f(vorticityProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
		gl.uniform1i(vorticityProgram.uniforms.uVelocity, velocity.read.attach(0));
		gl.uniform1i(vorticityProgram.uniforms.uCurl, curl.attach(1));
		gl.uniform1f(vorticityProgram.uniforms.curl, CURL);
		gl.uniform1f(vorticityProgram.uniforms.dt, dt);
		blit(velocity.write);
		velocity.swap();

		divergenceProgram.bind();
		gl.uniform2f(divergenceProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
		gl.uniform1i(divergenceProgram.uniforms.uVelocity, velocity.read.attach(0));
		blit(divergence);

		clearProgram.bind();
		gl.uniform1i(clearProgram.uniforms.uTexture, pressure.read.attach(0));
		gl.uniform1f(clearProgram.uniforms.value, PRESSURE);
		blit(pressure.write);
		pressure.swap();

		pressureProgram.bind();
		gl.uniform2f(pressureProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
		gl.uniform1i(pressureProgram.uniforms.uDivergence, divergence.attach(0));
		for (let i = 0; i < PRESSURE_ITERATIONS; i++) {
			gl.uniform1i(pressureProgram.uniforms.uPressure, pressure.read.attach(1));
			blit(pressure.write);
			pressure.swap();
		}

		gradienSubtractProgram.bind();
		gl.uniform2f(
			gradienSubtractProgram.uniforms.texelSize,
			velocity.texelSizeX,
			velocity.texelSizeY
		);
		gl.uniform1i(gradienSubtractProgram.uniforms.uPressure, pressure.read.attach(0));
		gl.uniform1i(gradienSubtractProgram.uniforms.uVelocity, velocity.read.attach(1));
		blit(velocity.write);
		velocity.swap();

		advectionProgram.bind();
		gl.uniform2f(advectionProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
		if (!ext.supportLinearFiltering)
			gl.uniform2f(
				advectionProgram.uniforms.dyeTexelSize,
				velocity.texelSizeX,
				velocity.texelSizeY
			);
		const velocityId = velocity.read.attach(0);
		gl.uniform1i(advectionProgram.uniforms.uVelocity, velocityId);
		gl.uniform1i(advectionProgram.uniforms.uSource, velocityId);
		gl.uniform1f(advectionProgram.uniforms.dt, dt);
		gl.uniform1f(advectionProgram.uniforms.dissipation, VELOCITY_DISSIPATION);
		blit(velocity.write);
		velocity.swap();

		if (!ext.supportLinearFiltering)
			gl.uniform2f(advectionProgram.uniforms.dyeTexelSize, dye.texelSizeX, dye.texelSizeY);
		gl.uniform1i(advectionProgram.uniforms.uVelocity, velocity.read.attach(0));
		gl.uniform1i(advectionProgram.uniforms.uSource, dye.read.attach(1));
		gl.uniform1f(advectionProgram.uniforms.dissipation, DENSITY_DISSIPATION);
		blit(dye.write);
		dye.swap();
	}

	function render(target) {
		if (BLOOM) applyBloom(dye.read, bloom);
		if (SUNRAYS) {
			applySunrays(dye.read, dye.write, sunrays);
			blur(sunrays, sunraysTemp, 1);
		}

		if (target == null || !TRANSPARENT) {
			gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
			gl.enable(gl.BLEND);
		} else {
			gl.disable(gl.BLEND);
		}

		if (!TRANSPARENT) drawColor(target, normalizeColor(BACK_COLOR));
		if (target == null && TRANSPARENT) drawCheckerboard(target);
		drawDisplay(target);
	}

	function drawColor(target, color) {
		colorProgram.bind();
		gl.uniform4f(colorProgram.uniforms.color, color.r, color.g, color.b, 1);
		blit(target);
	}

	function drawCheckerboard(target) {
		checkerboardProgram.bind();
		gl.uniform1f(checkerboardProgram.uniforms.aspectRatio, canvas.width / canvas.height);
		blit(target);
	}

	function drawDisplay(target) {
		const width = target == null ? gl.drawingBufferWidth : target.width;
		const height = target == null ? gl.drawingBufferHeight : target.height;

		displayMaterial.bind();
		if (SHADING) gl.uniform2f(displayMaterial.uniforms.texelSize, 1.0 / width, 1.0 / height);
		gl.uniform1i(displayMaterial.uniforms.uTexture, dye.read.attach(0));
		if (BLOOM) {
			gl.uniform1i(displayMaterial.uniforms.uBloom, bloom.attach(1));
			gl.uniform1i(displayMaterial.uniforms.uDithering, ditheringTexture.attach(2));
			const scale = getTextureScale(ditheringTexture, width, height);
			gl.uniform2f(displayMaterial.uniforms.ditherScale, scale.x, scale.y);
		}
		if (SUNRAYS) gl.uniform1i(displayMaterial.uniforms.uSunrays, sunrays.attach(3));
		blit(target);
	}

	function applyBloom(source, destination) {
		if (bloomFramebuffers.length < 2) return;

		let last = destination;

		gl.disable(gl.BLEND);
		bloomPrefilterProgram.bind();
		const knee = BLOOM_THRESHOLD * BLOOM_SOFT_KNEE + 0.0001;
		const curve0 = BLOOM_THRESHOLD - knee;
		const curve1 = knee * 2;
		const curve2 = 0.25 / knee;
		gl.uniform3f(bloomPrefilterProgram.uniforms.curve, curve0, curve1, curve2);
		gl.uniform1f(bloomPrefilterProgram.uniforms.threshold, BLOOM_THRESHOLD);
		gl.uniform1i(bloomPrefilterProgram.uniforms.uTexture, source.attach(0));
		blit(last);

		bloomBlurProgram.bind();
		for (let i = 0; i < bloomFramebuffers.length; i++) {
			const dest = bloomFramebuffers[i];
			gl.uniform2f(bloomBlurProgram.uniforms.texelSize, last.texelSizeX, last.texelSizeY);
			gl.uniform1i(bloomBlurProgram.uniforms.uTexture, last.attach(0));
			blit(dest);
			last = dest;
		}

		gl.blendFunc(gl.ONE, gl.ONE);
		gl.enable(gl.BLEND);

		for (let i = bloomFramebuffers.length - 2; i >= 0; i--) {
			const baseTex = bloomFramebuffers[i];
			gl.uniform2f(bloomBlurProgram.uniforms.texelSize, last.texelSizeX, last.texelSizeY);
			gl.uniform1i(bloomBlurProgram.uniforms.uTexture, last.attach(0));
			gl.viewport(0, 0, baseTex.width, baseTex.height);
			blit(baseTex);
			last = baseTex;
		}

		gl.disable(gl.BLEND);
		bloomFinalProgram.bind();
		gl.uniform2f(bloomFinalProgram.uniforms.texelSize, last.texelSizeX, last.texelSizeY);
		gl.uniform1i(bloomFinalProgram.uniforms.uTexture, last.attach(0));
		gl.uniform1f(bloomFinalProgram.uniforms.intensity, BLOOM_INTENSITY);
		blit(destination);
	}

	function applySunrays(source, mask, destination) {
		gl.disable(gl.BLEND);
		sunraysMaskProgram.bind();
		gl.uniform1i(sunraysMaskProgram.uniforms.uTexture, source.attach(0));
		blit(mask);

		sunraysProgram.bind();
		gl.uniform1f(sunraysProgram.uniforms.weight, SUNRAYS_WEIGHT);
		gl.uniform1i(sunraysProgram.uniforms.uTexture, mask.attach(0));
		blit(destination);
	}

	function blur(target, temp, iterations) {
		blurProgram.bind();
		for (let i = 0; i < iterations; i++) {
			gl.uniform2f(blurProgram.uniforms.texelSize, target.texelSizeX, 0.0);
			gl.uniform1i(blurProgram.uniforms.uTexture, target.attach(0));
			blit(temp);

			gl.uniform2f(blurProgram.uniforms.texelSize, 0.0, target.texelSizeY);
			gl.uniform1i(blurProgram.uniforms.uTexture, temp.attach(0));
			blit(target);
		}
	}

	function splatPointer(pointer) {
		const dx = pointer.deltaX * SPLAT_FORCE;
		const dy = pointer.deltaY * SPLAT_FORCE;
		splat(pointer.texcoordX, pointer.texcoordY, dx, dy, pointer.color);
	}

	function multipleSplats(amount) {
		for (let i = 0; i < amount; i++) {
			const color = generateColor();
			color.r *= 10.0;
			color.g *= 10.0;
			color.b *= 10.0;
			const x = Math.random();
			const y = Math.random();
			const dx = 1000 * (Math.random() - 0.5);
			const dy = 1000 * (Math.random() - 0.5);
			splat(x, y, dx, dy, color);
		}
	}

	function randomSplats() {
		splatStack.push(Math.trunc(Math.random() * 20) + 5);
	}

	function splat(x, y, dx, dy, color) {
		splatProgram.bind();
		gl.uniform1i(splatProgram.uniforms.uTarget, velocity.read.attach(0));
		gl.uniform1f(splatProgram.uniforms.aspectRatio, canvas.width / canvas.height);
		gl.uniform2f(splatProgram.uniforms.point, x, y);
		gl.uniform3f(splatProgram.uniforms.color, dx, dy, 0.0);
		gl.uniform1f(splatProgram.uniforms.radius, correctRadius(SPLAT_RADIUS / 100.0));
		blit(velocity.write);
		velocity.swap();

		gl.uniform1i(splatProgram.uniforms.uTarget, dye.read.attach(0));
		gl.uniform3f(splatProgram.uniforms.color, color.r, color.g, color.b);
		blit(dye.write);
		dye.swap();
	}

	function correctRadius(radius) {
		const aspectRatio = canvas.width / canvas.height;
		if (aspectRatio > 1) radius *= aspectRatio;
		return radius;
	}

	function getResolution(resolution) {
		let aspectRatio = gl.drawingBufferWidth / gl.drawingBufferHeight;
		if (aspectRatio < 1) aspectRatio = 1.0 / aspectRatio;

		const min = Math.round(resolution);
		const max = Math.round(resolution * aspectRatio);

		if (gl.drawingBufferWidth > gl.drawingBufferHeight) return { width: max, height: min };
		else return { width: min, height: max };
	}

	/** @type {HTMLCanvasElement} */
	let canvas;

	/** @type {WebGL2RenderingContext} */
	let gl;
	let ext;

	const pointers = [];
	const splatStack = [];

	let dye;
	let velocity;
	let divergence;
	let curl;
	let pressure;
	let bloom;
	let bloomFramebuffers = [];
	let sunrays;
	let sunraysTemp;

	let ditheringTexture;

	let blurProgram;
	let copyProgram;
	let clearProgram;
	let colorProgram;
	let checkerboardProgram;
	let bloomPrefilterProgram;
	let bloomBlurProgram;
	let bloomFinalProgram;
	let sunraysMaskProgram;
	let sunraysProgram;
	let splatProgram;
	let advectionProgram;
	let divergenceProgram;
	let curlProgram;
	let vorticityProgram;
	let pressureProgram;
	let gradienSubtractProgram;

	let displayMaterial;

	let lastUpdateTime;
	let colorUpdateTimer;

	/** @type {(target: any, clear?: boolean) => void} */
	let blit;

	onMount(() => {
		resizeCanvas();

		pointers.push(createPointer());

		// getWebGLContext
		const params = {
			alpha: true,
			depth: false,
			stencil: false,
			antialias: false,
			preserveDrawingBuffer: false
		};

		gl = /** @type {WebGL2RenderingContext} */ (canvas.getContext('webgl2', params));
		gl.getExtension('EXT_color_buffer_float');
		const supportLinearFiltering = gl.getExtension('OES_texture_float_linear');

		gl.clearColor(0.0, 0.0, 0.0, 1.0);

		const halfFloatTexType = gl.HALF_FLOAT;
		const formatRGBA = getSupportedFormat(gl, gl.RGBA16F, gl.RGBA, halfFloatTexType);
		const formatRG = getSupportedFormat(gl, gl.RG16F, gl.RG, halfFloatTexType);
		const formatR = getSupportedFormat(gl, gl.R16F, gl.RED, halfFloatTexType);

		ext = {
			formatRGBA,
			formatRG,
			formatR,
			halfFloatTexType,
			supportLinearFiltering
		};

		if (isMobile()) {
			DYE_RESOLUTION = 512;
		}
		if (!ext.supportLinearFiltering) {
			DYE_RESOLUTION = 512;
			SHADING = false;
			BLOOM = false;
			SUNRAYS = false;
		}

		const baseVertexShader = compileShader(gl.VERTEX_SHADER, shaders.baseVertexShader);
		const blurVertexShader = compileShader(gl.VERTEX_SHADER, shaders.blurVertexShader);

		const blurShader = compileShader(gl.FRAGMENT_SHADER, shaders.blurShader);
		const copyShader = compileShader(gl.FRAGMENT_SHADER, shaders.copyShader);
		const clearShader = compileShader(gl.FRAGMENT_SHADER, shaders.clearShader);
		const colorShader = compileShader(gl.FRAGMENT_SHADER, shaders.colorShader);
		const checkerboardShader = compileShader(gl.FRAGMENT_SHADER, shaders.checkerboardShader);
		const bloomPrefilterShader = compileShader(gl.FRAGMENT_SHADER, shaders.bloomPrefilterShader);
		const bloomBlurShader = compileShader(gl.FRAGMENT_SHADER, shaders.bloomBlurShader);
		const bloomFinalShader = compileShader(gl.FRAGMENT_SHADER, shaders.bloomFinalShader);
		const sunraysMaskShader = compileShader(gl.FRAGMENT_SHADER, shaders.sunraysMaskShader);
		const sunraysShader = compileShader(gl.FRAGMENT_SHADER, shaders.sunraysShader);
		const splatShader = compileShader(gl.FRAGMENT_SHADER, shaders.splatShader);
		const advectionShader = compileShader(
			gl.FRAGMENT_SHADER,
			shaders.advectionShader,
			ext.supportLinearFiltering ? null : ['MANUAL_FILTERING']
		);
		const divergenceShader = compileShader(gl.FRAGMENT_SHADER, shaders.divergenceShader);
		const curlShader = compileShader(gl.FRAGMENT_SHADER, shaders.curlShader);
		const vorticityShader = compileShader(gl.FRAGMENT_SHADER, shaders.vorticityShader);
		const pressureShader = compileShader(gl.FRAGMENT_SHADER, shaders.pressureShader);
		const gradientSubtractShader = compileShader(
			gl.FRAGMENT_SHADER,
			shaders.gradientSubtractShader
		);

		blit = createBlit(gl);

		ditheringTexture = createTextureAsync('LDR_LLL1_0.png');

		blurProgram = createProgram(blurVertexShader, blurShader);
		copyProgram = createProgram(baseVertexShader, copyShader);
		clearProgram = createProgram(baseVertexShader, clearShader);
		colorProgram = createProgram(baseVertexShader, colorShader);
		checkerboardProgram = createProgram(baseVertexShader, checkerboardShader);
		bloomPrefilterProgram = createProgram(baseVertexShader, bloomPrefilterShader);
		bloomBlurProgram = createProgram(baseVertexShader, bloomBlurShader);
		bloomFinalProgram = createProgram(baseVertexShader, bloomFinalShader);
		sunraysMaskProgram = createProgram(baseVertexShader, sunraysMaskShader);
		sunraysProgram = createProgram(baseVertexShader, sunraysShader);
		splatProgram = createProgram(baseVertexShader, splatShader);
		advectionProgram = createProgram(baseVertexShader, advectionShader);
		divergenceProgram = createProgram(baseVertexShader, divergenceShader);
		curlProgram = createProgram(baseVertexShader, curlShader);
		vorticityProgram = createProgram(baseVertexShader, vorticityShader);
		pressureProgram = createProgram(baseVertexShader, pressureShader);
		gradienSubtractProgram = createProgram(baseVertexShader, gradientSubtractShader);

		displayMaterial = createMaterial(baseVertexShader, shaders.displayShader);

		updateKeywords();
		initFramebuffers();
		multipleSplats(Math.trunc(Math.random() * 20) + 5);

		lastUpdateTime = Date.now();
		colorUpdateTimer = 0.0;
		update();
	});
</script>

<canvas
	bind:this={canvas}
	on:mousedown={(e) => {
		const posX = scaleByPixelRatio(e.offsetX);
		const posY = scaleByPixelRatio(e.offsetY);
		let pointer = pointers.find((p) => p.id == -1);
		if (pointer == null) pointer = createPointer();
		updatePointerDownData(pointer, -1, posX, posY);
	}}
	on:mousemove={(e) => {
		const pointer = pointers[0];
		if (!pointer.down) return;
		const posX = scaleByPixelRatio(e.offsetX);
		const posY = scaleByPixelRatio(e.offsetY);
		updatePointerMoveData(pointer, posX, posY);
	}}
	on:touchstart={(e) => {
		e.preventDefault();
		const touches = e.targetTouches;
		while (touches.length >= pointers.length) pointers.push(createPointer());
		for (let i = 0; i < touches.length; i++) {
			const posX = scaleByPixelRatio(touches[i].pageX);
			const posY = scaleByPixelRatio(touches[i].pageY);
			updatePointerDownData(pointers[i + 1], touches[i].identifier, posX, posY);
		}
	}}
	on:touchmove={(e) => {
		e.preventDefault();
		const touches = e.targetTouches;
		for (let i = 0; i < touches.length; i++) {
			const pointer = pointers[i + 1];
			if (!pointer.down) continue;
			const posX = scaleByPixelRatio(touches[i].pageX);
			const posY = scaleByPixelRatio(touches[i].pageY);
			updatePointerMoveData(pointer, posX, posY);
		}
	}}
/>

<svelte:window
	on:mouseup={() => (pointers[0].down = false)}
	on:touchend={(e) => {
		const touches = e.changedTouches;
		for (let i = 0; i < touches.length; i++) {
			const pointer = pointers.find((p) => p.id == touches[i].identifier);
			if (pointer == null) continue;
			pointer.down = false;
		}
	}}
	on:keydown={(e) => {
		if (e.code === 'KeyP') PAUSED = !PAUSED;
		if (e.key === ' ') splatStack.push(Math.trunc(Math.random() * 20) + 5);
	}}
/>

<style>
	canvas {
		width: 100%;
		height: 100%;
	}
</style>
