# blender.pyimport bpy
import random

# 1. LIMPIAR LA ESCENA ACTUAL
bpy.ops.wm.read_factory_settings(use_empty=True)

# 2. CONFIGURAR MOTOR DE RENDER Y POSPROCESADO (GLARE / BLOOM)
scene = bpy.context.scene
scene.render.engine = 'CYCLES' # O 'BLENDER_EEVEE_NEXT' para tiempo real
scene.cycles.samples = 64

# Activar nodos de composición para el efecto Bloom (Glare)
scene.use_nodes = True
comp_tree = scene.node_tree
comp_tree.nodes.clear()

render_layers = comp_tree.nodes.new('CompositorNodeRLayers')
glare_node = comp_tree.nodes.new('CompositorNodeGlare')
glare_node.glare_type = 'FOG_GLOW' # Simula resplandor cinematográfico
glare_node.threshold = 1.0
glare_node.size = 7

composite_out = comp_tree.nodes.new('CompositorNodeComposite')

comp_tree.links.new(render_layers.outputs['Image'], glare_node.inputs['Image'])
comp_tree.links.new(glare_node.outputs['Image'], composite_out.inputs['Image'])

# 3. CREAR MATERIAL PBR PROCEDURAL (METAL RUGOSO CON RELIEVE)
mat_metal = bpy.data.materials.new(name="Material_PBR_Metal")
mat_metal.use_nodes = True
nodes = mat_metal.node_tree.nodes
links = mat_metal.node_tree.links
nodes.clear()

# Nodos principales
output = nodes.new('ShaderNodeOutputMaterial')
bsdf = nodes.new('ShaderNodeBsdfPrincipled')
bsdf.inputs['Base Color'].default_value = (0.05, 0.25, 0.8, 1.0) # Azul metálico
bsdf.inputs['Metallic'].default_value = 0.9

# Textura de ruido para rugosidad y relieve
noise = nodes.new('ShaderNodeTexNoise')
noise.inputs['Scale'].default_value = 15.0
noise.inputs['Detail'].default_value = 8.0

# Rampa de color para controlar la rugosidad (Roughness)
ramp = nodes.new('ShaderNodeValToRGB')
ramp.color_ramp.elements[0].position = 0.3
ramp.color_ramp.elements[0].color = (0.15, 0.15, 0.15, 1.0) # Zonas pulidas
ramp.color_ramp.elements[1].position = 0.7
ramp.color_ramp.elements[1].color = (0.55, 0.55, 0.55, 1.0) # Zonas ásperas

# Nodo Bump para simular mapa de normales (relieve)
bump = nodes.new('ShaderNodeBump')
bump.inputs['Strength'].default_value = 0.2

# Conectar nodos del material
links.new(noise.outputs['Fac'], ramp.inputs['Fac'])
links.new(ramp.outputs['Color'], bsdf.inputs['Roughness'])
links.new(noise.outputs['Fac'], bump.inputs['Height'])
links.new(bump.outputs['Normal'], bsdf.inputs['Normal'])
links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])

# 4. CREAR MATERIAL EMISIVO (PARA ACTIVAR EL RESPLANDOR)
mat_neon = bpy.data.materials.new(name="Material_Neon")
mat_neon.use_nodes = True
n_nodes = mat_neon.node_tree.nodes
n_bsdf = n_nodes.get('Principled BSDF')
n_bsdf.inputs['Base Color'].default_value = (1.0, 0.05, 0.3, 1.0)
n_bsdf.inputs['Emission Color'].default_value = (1.0, 0.05, 0.3, 1.0)
n_bsdf.inputs['Emission Strength'].default_value = 8.0 # Intensidad alta

# 5. CREAR GEOMETRÍAS Y APLICAR MATERIALES
# Esfera central subdividida
bpy.ops.mesh.primitive_uv_sphere_add(segments=64, ring_count=32, radius=1.2, location=(0, 0, 1.5))
sphere = bpy.context.active_object
bpy.ops.object.shade_smooth()
sphere.data.materials.append(mat_metal)

# Aro de neón alrededor
bpy.ops.mesh.primitive_torus_add(major_segments=64, minor_segments=24, major_radius=2.0, minor_radius=0.05, location=(0, 0, 1.5), rotation=(0.6, 0.2, 0))
ring = bpy.context.active_object
bpy.ops.object.shade_smooth()
ring.data.materials.append(mat_neon)

# Suelo oscuro
bpy.ops.mesh.primitive_plane_add(size=30, location=(0, 0, 0))
floor = bpy.context.active_object
mat_floor = bpy.data.materials.new(name="Suelo_Oscuro")
mat_floor.use_nodes = True
mat_floor.node_tree.nodes['Principled BSDF'].inputs['Base Color'].default_value = (0.02, 0.02, 0.03, 1.0)
mat_floor.node_tree.nodes['Principled BSDF'].inputs['Roughness'].default_value = 0.4
floor.data.materials.append(mat_floor)

# 6. NIEBLA VOLUMÉTRICA EN EL MUNDO (ATMOSFERA)
world = bpy.data.worlds.new("Mundo_Volumetrico")
scene.world = world
world.use_nodes = True
w_nodes = world.node_tree.nodes
w_links = world.node_tree.links
w_nodes.clear()

w_out = w_nodes.new('ShaderNodeOutputWorld')
w_bg = w_nodes.new('ShaderNodeBackground')
w_bg.inputs['Color'].default_value = (0.01, 0.01, 0.02, 1.0)

w_vol = w_nodes.new('ShaderNodeVolumeScatter')
w_vol.inputs['Density'].default_value = 0.02 # Densidad de la niebla
w_vol.inputs['Anisotropy'].default_value = 0.6 # Dispersión de rayos de luz

w_links.new(w_bg.outputs['Background'], w_out.inputs['Surface'])
w_links.new(w_vol.outputs['Volume'], w_out.inputs['Volume'])

# 7. ILUMINACIÓN Y CÁMARA
bpy.ops.object.light_add(type='AREA', location=(4, -4, 5))
key_light = bpy.context.active_object
key_light.data.energy = 300
key_light.data.size = 2.0
key_light.data.color = (0.6, 0.9, 1.0)

# Apuntar luz al centro
track = key_light.constraints.new(type='TRACK_TO')
track.target = sphere
track.track_axis = 'TRACK_NEGATIVE_Z'
track.up_axis = 'UP_Y'

# Cámara con profundidad de campo (Depth of Field)
bpy.ops.object.camera_add(location=(0, -6.5, 2.5), rotation=(1.4, 0, 0))
cam = bpy.context.active_object
scene.camera = cam
cam.data.dof.use_dof = True
cam.data.dof.focus_object = sphere
cam.data.dof.aperture_fstop = 2.8

# 8. VFX: PARTÍCULAS FLOTANTES
for i in range(40):
    x, y, z = random.uniform(-4, 4), random.uniform(-4, 4), random.uniform(0.2, 4)
    bpy.ops.mesh.primitive_ico_sphere_add(subdivisions=1, radius=0.02, location=(x, y, z))
    part = bpy.context.active_object
    part.data.materials.append(mat_neon)
