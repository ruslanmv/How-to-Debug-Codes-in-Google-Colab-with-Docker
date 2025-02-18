## Debugging Python Code in Docker Containers with Gradio and Google Colab

Debugging code can be a real pain, especially when dealing with complex environments like Docker containers. But what if you could streamline the process and make it more interactive? In this blog post, we'll explore how to use Gradio and Google Colab to create a user-friendly debugging environment for Python code running in a Docker container.

### Why Docker and Colab?

Docker provides a consistent and isolated environment for your code, ensuring that it runs the same way regardless of the underlying system. Google Colab offers free access to powerful computing resources, including GPUs, making it an ideal platform for running computationally intensive tasks. Combining these two technologies gives you a robust and accessible platform for developing and debugging your Python code.

### Setting up the Environment

First, we'll need to install the necessary libraries and set up our Docker container in Google Colab. Here's the code to get started:

```python
!pip install udocker gradio
!udocker install
!udocker pull ubuntu:latest
!mkdir my_python_project
!udocker create --name my-debug-container ubuntu:latest
```

This code installs `udocker` (a tool for running Docker containers in user space) and `gradio` (a library for creating user interfaces), pulls the latest Ubuntu image, creates a directory for our project, and finally creates a Docker container named `my-debug-container`.

### Creating the Gradio Interface

Next, we'll create a Gradio interface that allows us to input our Python code, run it in the Docker container, and view the logs and terminal output. Here's the complete code:

```python
import gradio as gr
import subprocess
import os
import pty

def run_code_in_container(code, terminal_commands):
    # Write code to a file
    with open("my_python_project/user_script.py", "w") as f:
        f.write(code)

    try:
        # Start the container if it's not already running
        container_status = subprocess.run(['udocker', 'ps'], capture_output=True, text=True).stdout
        if 'my-debug-container' not in container_status:
            subprocess.run(['udocker', 'start', 'my-debug-container'])

        # Prepare the command to run in the container
        log_file_path = "/my_python_project/user_script.log"
        python_command = f"python3.9 -c \"import logging; logging.basicConfig(filename='{log_file_path}', level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s'); try: exec(open('/my_python_project/user_script.py').read()) except Exception as e: logging.exception('An error occurred: %s', e)\""

        # Create a pseudo-terminal
        master, slave = pty.openpty()

        # Run the command in the container with the pseudo-terminal
        container_process = subprocess.Popen(['udocker', 'exec', '-v', '/content/my_python_project:/my_python_project', 'my-debug-container', 'bash', '-c', python_command], stdin=slave, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)

        # Send terminal commands to the container
        if terminal_commands:
            for cmd in terminal_commands.splitlines():
                os.write(slave, (cmd + '\n').encode())

        # Capture output and errors (including from the pseudo-terminal)
        stdout, stderr = container_process.communicate()

        # Read and return the logs
        with open("my_python_project/user_script.log", "r") as log_file:
            logs = log_file.read()

        combined_output = stdout + stderr

        return combined_output, logs

    except Exception as e:
        return f"Error running code: {e}", ""

# Gradio interface
with gr.Blocks() as demo:
    code_input = gr.Code(label="Enter your Python code", language="python")
    terminal_input = gr.Textbox(label="Enter terminal commands (optional, one command per line)", lines=5)
    terminal_output = gr.Textbox(label="Terminal Output", lines=10)
    log_output = gr.Textbox(label="Log Output")
    run_button = gr.Button("Run Code")

    run_button.click(run_code_in_container, inputs=[code_input, terminal_input], outputs=[terminal_output, log_output])

demo.launch()
```

This code defines a function `run_code_in_container` that takes the Python code and any terminal commands as input, writes the code to a file, runs it in the Docker container, and captures the logs and terminal output. The Gradio interface provides input fields for the code and terminal commands, a button to run the code, and output boxes to display the results.

### Example Python Code with Errors

To demonstrate the debugging capabilities, let's use the following Python code with some intentional errors:

```python
import random

def buggy_function(x):
    if x > 5:
        return x / 0  # Division by zero error
    elif x < 0:
        raise ValueError("x must be non-negative") # Value error
    else:
        return random.randint(1, 10)  # Correct case

def main():
    try:
        result = buggy_function(7)
        print(f"Result: {result}")

        result = buggy_function(-2) # This will raise ValueError
        print(f"Result: {result}")

        result = buggy_function(3)
        print(f"Result: {result}")

    except Exception as e:
        print(f"An error occurred: {e}")

if __name__ == "__main__":
    main()
```

This code has a few issues:

- A `ZeroDivisionError` will occur when `x` is greater than 5.
- A `ValueError` will be raised when `x` is negative.

### Running the Code

Now, you can run this code in your Google Colab notebook. The Gradio interface will appear, allowing you to paste the example Python code and any terminal commands you want to execute in the container. Click the "Run Code" button, and the output will be displayed in the output boxes, showing the errors and logs.

### Benefits of this Approach

This approach offers several benefits:

* **Interactive Debugging:** You can easily experiment with different code snippets and see the results in real-time.
* **Terminal Access:** You have full access to the container's terminal, allowing you to run commands and inspect the environment.
* **Log Monitoring:** You can monitor the logs to see detailed information about the code's execution and any errors that occur.
* **User-Friendly Interface:** The Gradio interface makes the debugging process more accessible and intuitive.

### Conclusion

Debugging code in Docker containers can be challenging, but with the help of Gradio and Google Colab, you can create a more interactive and user-friendly environment. This approach allows you to experiment with your code, monitor its execution, and quickly identify and fix any errors. So, give it a try and see how it can improve your debugging workflow!
