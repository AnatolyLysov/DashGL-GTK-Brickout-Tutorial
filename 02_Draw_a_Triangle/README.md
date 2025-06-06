# Draw a Triangle

![bricks_002](https://user-images.githubusercontent.com/25621780/128594499-2228a4cd-81ca-4e41-8bf7-c15653df645f.png)

Once we have a window open, the next step is going to be a triangle. This step is quite a jump from simply 
opening a window as we will need to initialize OpenGL. We'll start with the main function, where we add a
GTKGlArea widget to our window. 

File: main.c
```c
int main(int argc, char *argv[]) {

	GtkWidget *window;
	GtkWidget *glArea;

	gtk_init(&argc, &argv);

	// Initialize Window

	window = gtk_window_new(GTK_WINDOW_TOPLEVEL);
	gtk_window_set_title(GTK_WINDOW(window), "Brickout Tutorial");
	gtk_window_set_position(GTK_WINDOW(window), GTK_WIN_POS_CENTER);
	gtk_window_set_default_size(GTK_WINDOW(window), 640, 480);
	gtk_window_set_type_hint(GTK_WINDOW(window), GDK_WINDOW_TYPE_HINT_UTILITY);
	g_signal_connect(window, "destroy", G_CALLBACK(gtk_main_quit), NULL);

	// Initialize GTK GL Area

	glArea = gtk_gl_area_new();
	gtk_widget_set_vexpand(glArea, TRUE);
	gtk_widget_set_hexpand(glArea, TRUE);
	g_signal_connect(glArea, "realize", G_CALLBACK(on_realize), NULL);
	g_signal_connect(glArea, "render", G_CALLBACK(on_render), NULL);
	gtk_container_add(GTK_CONTAINER(window), glArea);

	// Show widgets

	gtk_widget_show_all(window);
	gtk_main();

	return 0;

}
```

We first create our glArea widget. And then tell the widget to expand to the full width and height of our
640x480 window with _gtk_widget_set_vexpand_ and _gtk_widget_set_hexpand_. From there we need to add two
events, with callbacks. The first is "realize", which will be called when the GtkGLArea is ready to initialize
so we can set up OpenGL. We also need to add a "render" event, that will describe what we want to draw to the
screen when our OpenGL context is ready. And then we add our GtkGLArea widget to the window. 

From here let's take a look at the "realize" callback.

```c
static void on_realize(GtkGLArea *area) {

	// Debug Message

	g_print("on realize\n");

	gtk_gl_area_make_current(area);
	if(gtk_gl_area_get_error(area) != NULL) {
		fprintf(stderr, "Unknown error\n");
		return;
	}

	const GLubyte *renderer = glGetString(GL_RENDER);
	const GLubyte *version = glGetString(GL_VERSION);

	printf("Renderer: %s\n", renderer);
	printf("OpenGL version supported %s\n", version);

	glClearColor(1.0f, 1.0f, 1.0f, 1.0f);

	glGenVertexArrays(1, &vao);
	glBindVertexArray(vao);

	GLfloat triangle_vertices[] = {
		 0.0,  0.8,
		-0.8, -0.8,
		 0.8, -0.8
	};
	
	glGenBuffers(1, &vbo_triangle);
	glBindBuffer(GL_ARRAY_BUFFER, vbo_triangle);
	glBufferData(
		GL_ARRAY_BUFFER,
		sizeof(triangle_vertices),
		triangle_vertices,
		GL_STATIC_DRAW
	);

	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 0, 0);
	glEnableVertexAttribArray(0);
	glDisableVertexAttribArray(0);
	
	GLint compile_ok = GL_FALSE;
	GLint link_ok = GL_FALSE;

	const char *vs_source = 
	"#version 120\n"
	"attribute vec2 coord2d; \n"
	"void main (void) {\n"
	"	gl_Position = vec4(coord2d, 0.0, 1.0);\n"
	"}";

	const char *fs_source =
	"#version 120\n"
	"void main (void) {\n"
	"	gl_FragColor[0] = 0.0;\n"
	"	gl_FragColor[1] = 0.0;\n"
	"	gl_FragColor[2] = 1.0;\n"
	"}";

	GLuint fs = glCreateShader(GL_FRAGMENT_SHADER);
	glShaderSource(fs, 1, &fs_source, NULL);
	glCompileShader(fs);
	glGetShaderiv(fs, GL_COMPILE_STATUS, &compile_ok);
	if(!compile_ok) {
		fprintf(stderr, "Error in fragment shader\n");
		return;
	}

	GLuint vs = glCreateShader(GL_VERTEX_SHADER);
	glShaderSource(vs, 1, &vs_source, NULL);
	glCompileShader(vs);
	glGetShaderiv(vs, GL_COMPILE_STATUS, &compile_ok);
	if(!compile_ok) {
		fprintf(stderr, "Error in vertex shader\n");
		return;
	}

	program = glCreateProgram();
	glAttachShader(program, vs);
	glAttachShader(program, fs);
	glLinkProgram(program);
	glGetProgramiv(program, GL_LINK_STATUS, &link_ok);
	if(!link_ok) {
		fprintf(stderr, "Error when linking program\n");
		return;
	}

	const char *attribute_name = "coord2d";
	attribute_coord2d = glGetAttribLocation(program, attribute_name);
	if(attribute_coord2d == -1) {
		fprintf(stderr, "Could not bind attribute %s\n", attribute_name);
		return;
	}
}
```

And I'll try and quickly cover what's going here. This should be thought of more 
as a "Hello World" aspect of OpenGL to get it started, you don't need to completely
understand everything upfront, as the familiarity will come with more use. 

First we call _gtk_gl_area_make_current_ which will make sure the OpenGL 
context uses our GtkGLArea widget. Then for a check we see which versions of 
OpenGL are supported. From there we set the clearColor to white, so that 
when we render, something to the screen it will be on a white background.

From there we set up what are called "Vertex Array Object" or vao. These are
required from OpenGL 3.0 and onward, which is what GTKGLArea uses. And these 
are more of an optimization, when drawing more complex scenes. We will set one up, 
and then only use that one, so just think of this as syntax for now. 

From there we set up the coorinates for our triangle. And then we set up our shaders.
Shaders are small programs that run on the GPU and tell the GPU what to do with the 
information that we pass into it. We have vertex shaders, which desribe where something
should be rendered, and fragment shaders that describe what color something will be.
We will cover this information in more detail in later steps. 

File: main.c
```c
static void on_render(GtkGLArea *area, GdkGLContext *context) {

	g_print("on render\n");

	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

	glUseProgram(program);

	glBindVertexArray(vao);
	glEnableVertexAttribArray(attribute_coord2d);

	glBindBuffer(GL_ARRAY_BUFFER, vbo_triangle);
	glVertexAttribPointer(
		attribute_coord2d,
		2,
		GL_FLOAT,
		GL_FALSE,
		0,
		0
	);

	glDrawArrays(GL_TRIANGLES, 0, 3);
	glDisableVertexAttribArray(attribute_coord2d);

}
```

And last is the render callback. First we clear everything to a white background.
Then we call tell OpenGL we want to use our shader program. We tell the shader
program how to interpret the 2d coordinates that we pass into it. And then we
call _glDrawArrays_ to draw our triangle to the screen. 

We compile with:
```
$ gcc `pkg-config --cflags gtk+-3.0` main.c `pkg-config --libs gtk+-3.0` -lepoxy
```

And then run our program with:
```
$ ./a.out
```

```
BITO
GTK (GIMP Toolkit) is an open-source, cross-platform widget toolkit primarily used for developing graphical user interfaces (GUIs). It provides a set of tools for creating windows, buttons, text boxes, and other interface components, making it easier for developers to build feature-rich desktop applications. Initially developed for the GNU Image Manipulation Program (GIMP), GTK has grown popular for creating applications on Linux and other systems, and it supports multiple programming languages through various bindings.

//////////////////////////////////////////////////////////////////////////////

GTK API is a robust set of functions and tools for creating graphical user interfaces (GUIs). Here’s a brief overview:

- **Widget Toolkit:** Provides a range of customizable widgets (buttons, labels, text boxes, etc.) that developers use to build complex interfaces.
- **Event-Driven Programming:** Uses a signal and callback system where application behavior is defined by responding to user events (clicks, key presses, etc.).
- **Cross-Platform:** Although originally designed for Linux, GTK supports multiple operating systems such as Windows and macOS.
- **Language Bindings:** Primarily written in C, GTK offers bindings for various programming languages (like Python, C++, and JavaScript), allowing broader use.
- **Theming and Accessibility:** Supports theming for customizing interface appearances and includes accessibility features to ensure applications are usable by everyone.

In summary, the GTK API is designed to simplify the process of building powerful, interactive GUIs across different platforms.

///////////////////////////////////////////////////////////////////////////////

# Code Summary

This code creates a GTK window with an area for OpenGL rendering. In that area, it sets up a simple OpenGL scene to draw a blue triangle on a white background. This involves initializing GTK and an OpenGL context, creating the necessary vertex data (a single triangle), compiling shaders (vertex and fragment shaders), and finally rendering the triangle when required.

# Step-by-Step Explanation

## 1. Main Function Setup

- **GTK Initialization and Window Creation**
  - The program starts by calling `gtk_init(&argc, &argv)` to initialize the GTK library.
  - A new window is created using `gtk_window_new(GTK_WINDOW_TOPLEVEL)`.
  - Window properties are set:
    - Title: `"Brickout Tutorial"`.
    - Window position is set to the center of the screen.
    - Default size of 640x480 pixels.
    - The window type is set as a utility to give an appropriate hint to the window manager.
  - A signal handler is connected to the "destroy" event so that when the window is closed, the main GTK loop quits.

- **Creating and Configuring the OpenGL Area (GtkGLArea)**
  - A `GtkGLArea` widget is created using `gtk_gl_area_new()`.
  - The widget is configured to expand in both horizontal and vertical directions within the window by using `gtk_widget_set_vexpand` and `gtk_widget_set_hexpand`.
  - Two important signal callbacks are connected:
    - **"realize" callback**: Triggered when the GTK GL area is ready to set up the OpenGL context. It calls the `on_realize` function.
    - **"render" callback**: Triggered when the GL area needs to be rendered, which calls the `on_render` function.
  - The OpenGL area is added to the window container with `gtk_container_add`.

- **Starting the GTK Main Loop**
  - `gtk_widget_show_all(window)` displays the window and all its child widgets.
  - Finally, `gtk_main()` starts the GTK main event loop, waiting for events such as rendering or window actions.

## 2. on_realize: OpenGL Initialization

- **Making the OpenGL Context Current**
  - `gtk_gl_area_make_current(area)` activates the OpenGL context that is associated with the GtkGLArea widget.
  - It checks for any OpenGL related errors after making the context current.

- **Retrieving OpenGL Renderer Information**
  - The code retrieves strings indicating which renderer is used and the supported OpenGL version using `glGetString`.
  - These details are printed to the console mainly for debugging purposes.

- **Setting a Clear Color**
  - `glClearColor` is used to set the background color that OpenGL will use when clearing the screen. Here, it is set to white (`1.0f, 1.0f, 1.0f, 1.0f`).

- **Creating a Vertex Array Object (VAO)**
  - A single VAO is generated and bound. VAOs store the state needed for rendering (like how vertex attributes are stored).
  
- **Setting up Triangle Vertex Data**
  - An array `triangle_vertices` is defined with 2D coordinates for each vertex of a triangle:
    - Top vertex: `(0.0, 0.8)`
    - Bottom left: `(-0.8, -0.8)`
    - Bottom right: `(0.8, -0.8)`
  - A Vertex Buffer Object (VBO) is generated and bound.
  - The vertex data is then copied to the GPU memory using `glBufferData`.

- **Configuring Vertex Attributes**
  - The code specifies how the vertex data is laid out using `glVertexAttribPointer` and initially enables and then disables the vertex attribute array. This configuration will later be re-enabled during rendering.

- **Compiling the Shaders**
  - **Fragment Shader:**
    - A source code string is defined to set the output color of the triangle to blue.
    - The shader is created, its source is loaded, and then it is compiled.
    - After compilation, it checks if the shader compiled correctly.
  
  - **Vertex Shader:**
    - A source code string is defined that takes input 2D coordinates (attribute `coord2d`) and converts them into a 4D position vector required by the OpenGL pipeline.
    - Similarly, the vertex shader is created, compiled, and checked for errors.
  
- **Linking the Shader Program**
  - A shader program is created, and both compiled vertex and fragment shaders are attached.
  - The program is linked, and there is an error check to ensure the linking was successful.
  
- **Binding the Vertex Attribute**
  - The attribute location for `coord2d` in the shader program is obtained using `glGetAttribLocation`.
  - If this attribute is not found (`-1`), an error is reported.

## 3. on_render: Drawing the Triangle

- **Clearing the Screen**
  - The `glClear` function is called with `GL_COLOR_BUFFER_BIT` and `GL_DEPTH_BUFFER_BIT` to clear all previous frame data. This results in a clean white background as set earlier.

- **Activating the Shader Program**
  - `glUseProgram(program)` tells OpenGL to use the previously linked shader program for subsequent drawing operations.

- **Binding the Vertex Data**
  - The previously generated VAO is bound.
  - The vertex attribute array for the triangle coordinates is enabled using the attribute location (`attribute_coord2d`) that was obtained earlier.
  - The VBO with the triangle's vertex data is re-bound and its vertex pointer is defined again using `glVertexAttribPointer`.

- **Rendering the Triangle**
  - `glDrawArrays` with the argument `GL_TRIANGLES` is called, which instructs OpenGL to draw triangles using the vertex data. Here, it draws one triangle made of 3 vertices.
  - After drawing, the vertex attribute array is disabled.

## 4. Compilation and Execution

- The code is compiled using GCC along with the necessary flags for GTK+ and linking with `-lepoxy` (a library that aids with function pointer management for OpenGL):
  - Example command:  
    gcc `pkg-config --cflags gtk+-3.0` main.c `pkg-config --libs gtk+-3.0` -lepoxy
- The compiled program is then executed (e.g., `./a.out`), which opens the window and displays the triangle.

# Conclusion

This code is a simple introduction to using GTK and OpenGL together. It initializes a GTK window with an OpenGL drawing area, sets up a basic OpenGL environment with a white background, configures a blue triangle using vertex and fragment shaders, and finally renders the triangle on the screen within the GTK application lifecycle.


akl(it's me) 
gcc -Wall -g `pkg-config --cflags gtk+-3.0` main.c `pkg-config --libs gtk+-3.0` -lepoxy


```