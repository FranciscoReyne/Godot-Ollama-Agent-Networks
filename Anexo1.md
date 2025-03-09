Voy a explicar en detalle los pasos exactos para configurar el área de trabajo con el SubViewport y todos los elementos necesarios:

## Configuración detallada del área de trabajo con SubViewport

1. **Crear y configurar el SubViewport:**
   - En tu escena principal, añade un nodo `SubViewportContainer`
   - Haz clic derecho en el `SubViewportContainer` → Añadir Hijo → `SubViewport`
   - Selecciona el `SubViewportContainer` y en el Inspector:
     - Activa "Stretch" para que se ajuste al tamaño del contenedor
     - Establece "Stretch Mode" a "viewport"

2. **Configurar la escena 2D dentro del SubViewport:**
   - Selecciona el nodo `SubViewport` que acabas de crear
   - Haz clic derecho → Añadir Hijo → `Node2D` (llámalo "Workspace")
   - Este `Node2D` será el contenedor de todos tus nodos de agentes

3. **Añadir script al nodo Workspace:**
   - Selecciona el nodo "Workspace"
   - Ve a la pestaña "Script" en el Inspector y crea un nuevo script llamado "WorkspaceController.gd"
   - Este script manejará todas las interacciones con el área de trabajo

4. **Implementación del script WorkspaceController.gd:**

```gdscript
# WorkspaceController.gd
extends Node2D

# Referencia al editor principal
@onready var editor = get_node("/root/NeuralEditor")

# Variables para manejar la interacción
var dragging_canvas = false
var drag_start_position = Vector2.ZERO
var zoom_level = 1.0
var selected_agent = null
var dragging_agent = false
var creating_connection = false
var connection_start_agent = null
var connection_preview_line = null

# Escena del nodo agente que cargaremos
var agent_scene = preload("res://scenes/AgentNode.tscn")

# Lista de todos los agentes y conexiones
var agents = []
var connections = []

func _ready():
    # Inicializar la línea de vista previa para conexiones
    connection_preview_line = Line2D.new()
    connection_preview_line.width = 2.0
    connection_preview_line.default_color = Color(1, 1, 1, 0.5)
    connection_preview_line.visible = false
    add_child(connection_preview_line)

func _unhandled_input(event):
    # Manejar zoom con la rueda del ratón
    if event is InputEventMouseButton:
        if event.button_index == MOUSE_BUTTON_WHEEL_UP:
            zoom_in()
            get_viewport().set_input_as_handled()
        elif event.button_index == MOUSE_BUTTON_WHEEL_DOWN:
            zoom_out()
            get_viewport().set_input_as_handled()
        # Iniciar arrastre del canvas con el botón central
        elif event.button_index == MOUSE_BUTTON_MIDDLE:
            if event.pressed:
                dragging_canvas = true
                drag_start_position = event.position
            else:
                dragging_canvas = false
            get_viewport().set_input_as_handled()
        # Clic izquierdo - seleccionar o crear
        elif event.button_index == MOUSE_BUTTON_LEFT:
            if event.pressed:
                if editor.current_tool == "create_agent":
                    create_agent_at_position(event.global_position)
                    get_viewport().set_input_as_handled()
                elif creating_connection:
                    handle_connection_interaction(event.global_position)
                    get_viewport().set_input_as_handled()
                else:
                    select_agent_at_position(event.global_position)
    
    # Manejar arrastre del canvas
    if event is InputEventMouseMotion:
        if dragging_canvas:
            position += (event.position - drag_start_position) * (1.0 / zoom_level)
            drag_start_position = event.position
            get_viewport().set_input_as_handled()
        elif dragging_agent and selected_agent != null:
            selected_agent.position = (get_global_mouse_position() - position) / zoom_level
            update_connections()
            get_viewport().set_input_as_handled()
        elif creating_connection and connection_start_agent != null:
            update_connection_preview()
            get_viewport().set_input_as_handled()

func zoom_in():
    zoom_level = min(zoom_level * 1.1, 3.0)
    scale = Vector2(zoom_level, zoom_level)

func zoom_out():
    zoom_level = max(zoom_level / 1.1, 0.3)
    scale = Vector2(zoom_level, zoom_level)

func create_agent_at_position(global_pos):
    var local_pos = (global_pos - position) / zoom_level
    
    # Crear instancia de AgentNode
    var new_agent = agent_scene.instantiate()
    add_child(new_agent)
    new_agent.position = local_pos
    
    # Configurar tipo según la herramienta actual
    var agent_type = editor.current_tool.replace("create_", "")
    new_agent.setup_agent(agent_type, "Agente " + str(agents.size() + 1))
    
    # Añadir a la lista
    agents.append(new_agent)
    
    # Notificar al editor que se seleccionó un agente
    select_agent(new_agent)
    
    # Volver automáticamente a la herramienta de selección
    editor.set_tool("select")

func select_agent_at_position(global_pos):
    var local_pos = (global_pos - position) / zoom_level
    
    # Deseleccionar el agente actual
    if selected_agent != null:
        selected_agent.set_selected(false)
        selected_agent = null
    
    # Buscar si hay un agente en esa posición
    for agent in agents:
        if agent.get_rect().has_point(local_pos - agent.position):
            select_agent(agent)
            dragging_agent = true
            break
    
    # Actualizar inspector
    editor.update_inspector()

func select_agent(agent):
    # Deseleccionar el agente actual
    if selected_agent != null:
        selected_agent.set_selected(false)
    
    # Seleccionar el nuevo agente
    selected_agent = agent
    selected_agent.set_selected(true)
    
    # Actualizar inspector
    editor.update_inspector(agent.get_data())

func start_connection(agent):
    creating_connection = true
    connection_start_agent = agent
    connection_preview_line.visible = true
    connection_preview_line.clear_points()
    connection_preview_line.add_point(connection_start_agent.get_output_pin_position())
    connection_preview_line.add_point(connection_start_agent.get_output_pin_position())

func update_connection_preview():
    if connection_preview_line.visible and connection_start_agent != null:
        connection_preview_line.set_point_position(0, connection_start_agent.get_output_pin_position())
        connection_preview_line.set_point_position(1, get_global_mouse_position())

func handle_connection_interaction(global_pos):
    var local_pos = (global_pos - position) / zoom_level
    
    # Si no hay agente inicial, buscamos uno para iniciar la conexión
    if connection_start_agent == null:
        for agent in agents:
            if agent.is_output_pin_at(local_pos - agent.position):
                start_connection(agent)
                return
    
    # Si ya hay un agente inicial, buscamos uno final
    else:
        for agent in agents:
            if agent != connection_start_agent and agent.is_input_pin_at(local_pos - agent.position):
                create_connection(connection_start_agent, agent)
                # Reiniciar estado
                connection_start_agent = null
                creating_connection = false
                connection_preview_line.visible = false
                return
        
        # Si se hizo clic pero no en un pin de entrada, cancelar
        connection_start_agent = null
        creating_connection = false
        connection_preview_line.visible = false
    
    editor.set_tool("select")

func create_connection(from_agent, to_agent):
    # Verificar que no exista ya una conexión entre estos agentes
    for conn in connections:
        if conn.from_agent == from_agent and conn.to_agent == to_agent:
            return
    
    # Crear la conexión
    var connection = {
        "from_agent": from_agent,
        "to_agent": to_agent,
        "line": Line2D.new(),
        "weight": 1.0
    }
    
    # Configurar la línea visual
    connection.line.width = 2.0
    connection.line.default_color = Color(0.0, 0.8, 1.0)
    add_child(connection.line)
    
    # Actualizar puntos de la línea
    connection.line.clear_points()
    connection.line.add_point(from_agent.get_output_pin_position())
    connection.line.add_point(to_agent.get_input_pin_position())
    
    # Añadir a la lista
    connections.append(connection)

func update_connections():
    for conn in connections:
        conn.line.clear_points()
        conn.line.add_point(conn.from_agent.get_output_pin_position())
        conn.line.add_point(conn.to_agent.get_input_pin_position())

func get_network_data():
    var data = {
        "agents": [],
        "connections": []
    }
    
    # Recopilar datos de los agentes
    for agent in agents:
        data.agents.append(agent.get_data())
    
    # Recopilar datos de las conexiones
    for conn in connections:
        data.connections.append({
            "from_id": agents.find(conn.from_agent),
            "to_id": agents.find(conn.to_agent),
            "weight": conn.weight
        })
    
    return data
```

5. **Creación del nodo AgentNode:**
   - Crea una nueva escena con un nodo raíz de tipo `Node2D`
   - Guárdala como "scenes/AgentNode.tscn"
   - Añade los siguientes nodos como hijos:
     - Un `ColorRect` para el fondo
     - Un `Label` para el nombre
     - Dos `TextureRect` para los pines de entrada y salida

6. **Script para el AgentNode:**
   - Crea un script "AgentNode.gd" y asígnalo al nodo raíz de la escena AgentNode

```gdscript
# AgentNode.gd
extends Node2D

signal pin_clicked(agent, is_output)

var agent_type = "Proceso"
var agent_name = "Agente"
var prompt_template = ""
var model = "llama3"
var api_endpoint = "http://localhost:11434/api/generate"
var activation_threshold = 0.5

var selected = false

func _ready():
    update_appearance()

func setup_agent(type, name):
    agent_type = type
    agent_name = name
    update_appearance()

func update_appearance():
    # Actualizar el color según el tipo
    var color = Color.WHITE
    match agent_type:
        "Sensor": color = Color(0.2, 0.8, 0.2)
        "Proceso": color = Color(0.2, 0.5, 0.9)
        "Decisión": color = Color(0.9, 0.3, 0.3)
    
    $Background.color = color
    $NameLabel.text = agent_name
    
    # Mostrar/ocultar pines según el tipo
    $InputPin.visible = agent_type != "Sensor"
    $OutputPin.visible = agent_type != "Decisión"

func get_rect():
    return Rect2(-75, -30, 150, 60)

func set_selected(is_selected):
    selected = is_selected
    $SelectionBorder.visible = selected

func is_output_pin_at(local_pos):
    if !$OutputPin.visible:
        return false
    return $OutputPin.get_rect().has_point(local_pos - $OutputPin.position)

func is_input_pin_at(local_pos):
    if !$InputPin.visible:
        return false
    return $InputPin.get_rect().has_point(local_pos - $InputPin.position)

func get_output_pin_position():
    return global_position + $OutputPin.position

func get_input_pin_position():
    return global_position + $InputPin.position

func get_data():
    return {
        "agent_name": agent_name,
        "agent_type": agent_type,
        "prompt_template": prompt_template,
        "model": model,
        "api_endpoint": api_endpoint,
        "activation_threshold": activation_threshold,
        "position": position
    }

func _on_InputPin_gui_input(event):
    if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
        emit_signal("pin_clicked", self, false)

func _on_OutputPin_gui_input(event):
    if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
        emit_signal("pin_clicked", self, true)
```

## Implementación del Inspector de Propiedades

1. **Crear el panel de inspector:**
   - En tu escena principal, añade un `PanelContainer` a la derecha
   - Dentro del PanelContainer, añade un `VBoxContainer`

2. **Añadir campos de edición:**
   - Dentro del VBoxContainer, añade los siguientes elementos:
     - Un `Label` con texto "Inspector de Propiedades"
     - Un `GridContainer` con 2 columnas para organizar campos y etiquetas

3. **Para cada propiedad, añade:**
   - Un `Label` con el nombre de la propiedad
   - El control apropiado para editar su valor:
     - `LineEdit` para Nombre, Modelo, Endpoint API
     - `OptionButton` para Tipo (con opciones "Sensor", "Proceso", "Decisión")
     - `TextEdit` para Plantilla de prompt
     - `SpinBox` para Umbral de activación

4. **Implementar la actualización del inspector:**
   - Añade este método a tu script NeuralEditor.gd:

```gdscript
func update_inspector(agent_data = null):
    if agent_data == null:
        # Desactivar todos los campos si no hay agente seleccionado
        $RightPanel/Inspector.visible = false
        return
    
    # Mostrar el inspector
    $RightPanel/Inspector.visible = true
    
    # Actualizar los campos
    $RightPanel/Inspector/Grid/NameEdit.text = agent_data.agent_name
    
    # Actualizar tipo en el OptionButton
    var type_option = $RightPanel/Inspector/Grid/TypeOption
    for i in range(type_option.get_item_count()):
        if type_option.get_item_text(i) == agent_data.agent_type:
            type_option.select(i)
            break
    
    $RightPanel/Inspector/Grid/PromptEdit.text = agent_data.prompt_template
    $RightPanel/Inspector/Grid/ModelEdit.text = agent_data.model
    $RightPanel/Inspector/Grid/EndpointEdit.text = agent_data.api_endpoint
    $RightPanel/Inspector/Grid/ThresholdSpin.value = agent_data.activation_threshold

# Conectar señales de cambio de valores
func _connect_inspector_signals():
    $RightPanel/Inspector/Grid/NameEdit.connect("text_changed", self, "_on_inspector_name_changed")
    $RightPanel/Inspector/Grid/TypeOption.connect("item_selected", self, "_on_inspector_type_changed")
    $RightPanel/Inspector/Grid/PromptEdit.connect("text_changed", self, "_on_inspector_prompt_changed")
    $RightPanel/Inspector/Grid/ModelEdit.connect("text_changed", self, "_on_inspector_model_changed")
    $RightPanel/Inspector/Grid/EndpointEdit.connect("text_changed", self, "_on_inspector_endpoint_changed")
    $RightPanel/Inspector/Grid/ThresholdSpin.connect("value_changed", self, "_on_inspector_threshold_changed")

# Métodos para actualizar el agente seleccionado
func _on_inspector_name_changed(new_text):
    if $Workspace.selected_agent != null:
        $Workspace.selected_agent.agent_name = new_text
        $Workspace.selected_agent.update_appearance()

func _on_inspector_type_changed(index):
    if $Workspace.selected_agent != null:
        var type = $RightPanel/Inspector/Grid/TypeOption.get_item_text(index)
        $Workspace.selected_agent.agent_type = type
        $Workspace.selected_agent.update_appearance()
        $Workspace.update_connections()
```

## Estructura del proyecto completa

Ahora deberías tener la siguiente estructura:

1. **Escena principal (NeuralEditor.tscn):**
   - HBoxContainer (raíz)
     - VBoxContainer (panel izquierdo - botones)
     - SubViewportContainer (área central)
       - SubViewport
         - Node2D (Workspace)
     - PanelContainer (panel derecho - inspector)
       - VBoxContainer
         - Label ("Inspector de Propiedades")
         - GridContainer (campos de edición)

2. **Escena de agente (AgentNode.tscn):**
   - Node2D (raíz)
     - ColorRect (fondo)
     - Label (nombre)
     - TextureRect (pin de entrada)
     - TextureRect (pin de salida)
     - ColorRect (borde de selección)

Con esta implementación detallada, tendrás un editor visual completo donde puedes:
- Crear agentes de diferentes tipos arrastrándolos al área de trabajo
- Seleccionar y mover agentes
- Hacer zoom y panorámica del área de trabajo
- Crear conexiones entre agentes
- Editar las propiedades de cada agente en el inspector

