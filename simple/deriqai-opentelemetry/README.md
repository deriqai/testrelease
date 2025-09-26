 # deriqai-opentelemetry Library Documentation
 
 This document provides instructions on how to deploy the `deriqai-opentelemetry` package and how to use its API for instrumenting your Python applications.
 
 ## 1. Deployment and Installation
 
 The `deriqai-opentelemetry` package is hosted on a custom Python package index. To install it, you need to configure your package manager (`pip` or `uv`) to use this index.
 
 **Index URL**: `https://deriqai.github.io/testrelease/simple/`
 
 ### Using `pip`
 
 You can install the package using `pip` by providing the extra index URL via the command line.
 
 ```bash
 pip install deriqai-opentelemetry --extra-index-url https://deriqai.github.io/testrelease/simple/
 ```
 
 To make this permanent for your project, you can add it to your `requirements.txt`:
 
 ```plaintext
 # requirements.txt
 --extra-index-url https://deriqai.github.io/testrelease/simple/
 deriqai-opentelemetry>=0.0.1
 ```
 
 Then install using:
 ```bash
 pip install -r requirements.txt
 ```
 
 ### Using `uv`
 
 If you are using `uv` as your package manager, you can configure the custom index in your `pyproject.toml` file. This is the recommended approach for projects managed with `uv`.
 
 ```toml
 # pyproject.toml
 
 [project]
 name = "your-project-name"
 # ...
 dependencies = [
     "deriqai-opentelemetry>=0.0.1",
     # other dependencies...
 ]
 
 # Define the custom index
 [[tool.uv.index]]
 name = "deriqai-testrelease"
 url = "https://deriqai.github.io/testrelease/simple/"
 
 # Map the package to the custom index
 [tool.uv.sources]
 deriqai-opentelemetry = { index = "deriqai-testrelease" }
 ```
 
 After configuring `pyproject.toml`, you can install dependencies as usual:
 
 ```bash
 uv pip install -e .
 # or
 uv sync
 ```
 ## 2. Import Style Guide

To ensure consistency across projects and documentation, we recommend using a single official alias for `deriqai-opentelemetry`. This prevents confusion with upstream OpenTelemetry modules and makes it immediately clear that the functionality originates from `deriqai`.

## Recommended Alias
 ```python
 from deriqai import opentelemetry as dqotel
 ```
- **`dqotel`** makes it clear that this module comes from `deriqai` and relates to OpenTelemetry.
- This avoids naming collisions and follows Python conventions of lowercase aliases.
- Prefixing with `dq` highlights that it originates from `deriqai`.
  
 ### General Guidance
 
- Keep aliases lowercase.
- Use `dq` as a prefix when possible to indicate `deriqai`.
 
 ## 3. API Usage Examples
 
 The `deriqai-opentelemetry` library simplifies instrumenting your application for observability. The core of the library is the `configure()` function, which sets up OpenTelemetry tracing and logging based on your application type.

 You should call this function as early as possible in your application's startup lifecycle.
 
 ### Example 1: Standalone Python Script
 
 For a basic Python script, use `AppType.OTEL_CORE`. This sets up the OpenTelemetry SDK for tracing and logging without any framework-specific integrations.
 
 ```python
 # main.py
 import logging
 from deriqai import opentelemetry as dqotel
 from opentelemetry import trace

 def setup_observability():
     """Configures DeriQAI OpenTelemetry for the application."""
     # 1. Configure DeriQAI OpenTelemetry
     dqotel.configure(
         app_type=dqotel.AppType.OTEL_CORE,
         resource={"service.name": "my-standalone-app", "service.version": "1.10.0", "deployment.environment": "production"},
         level=logging.INFO
     )

 setup_observability()

 # 2. Get a logger and tracer after configuration
 main_logger = logging.getLogger("my_app")
 tracer = trace.get_tracer(__name__)

 def main():
     with tracer.start_as_current_span("main-operation") as span:
         # Use 'extra', an optional parameter, for application-specific context.
         # This data will be attached to the log record for better observability.
         log_context = {"processing.mode": "batch"}
         main_logger.info("Hello from my standalone app!", extra=log_context)

         # Your application logic here
 if __name__ == "__main__":
     main()
 ```
 
 ### Example 2: Flask Application
 
 To instrument a Flask application, pass the Flask `app` object and use `AppType.FLASK`. The library will automatically instrument Flask to create spans for incoming requests.
 
 ```python
 # app.py
 import logging
 from flask import Flask
 from deriqai import opentelemetry as dqotel

 def setup_observability(flask_app: Flask):
     """Configures DeriQAI OpenTelemetry for the Flask application."""
     # 1. Configure DeriQAI for Flask
     dqotel.configure(
         app=flask_app,
         app_type=dqotel.AppType.FLASK,
         resource={"service.name": "my-flask-app", "service.version": "1.2.3", "deployment.environment": "production"},
         level=logging.INFO
     )

 app = Flask(__name__)
 setup_observability(app)

 flask_logger = logging.getLogger("flask_app")

 @app.route('/')
 def index():
     flask_logger.info("Received a request to the index route.")
     # Use 'extra', an optional parameter, to add business-logic context, like a customer tier or feature flag.
     log_context = {"customer.tier": "premium", "feature.flag.new_ui": True}
     flask_logger.info("Received a request to the index route.", extra=log_context)

     return "Request received!"
 if __name__ == '__main__':
     app.run(debug=False, port=4000)
 ```
 
 ### Example 3: Combined Flask + Celery OpenTelemetry Example
 This example demonstrates how to instrument a distributed application consisting of a Flask web server that dispatches tasks and a Celery worker that executes them. deriqai-opentelemetry is configured in three places to ensure the trace context is propagated across the entire system.

 - `tasks.py`: Defines the shared Celery application and task.
 - `flask_producer_app.py`: The Flask server that acts as the task producer.
 - `celery_worker.py`: The Celery worker that consumes and executes the tasks.

 ### 1. Task Definition (`tasks.py`)
 This file is shared by both the Flask producer and the Celery worker. It defines the Celery application and the background task.
 ```python
    # tasks.py
    import logging
    from celery import Celery

    # Initialize the Celery app, using RabbitMQ as the broker.
    celery_app = Celery('tasks', broker='pyamqp://guest@localhost//')
    celery_logger = logging.getLogger("celery_task")

    @celery_app.task
    def my_background_task(x, y):
        """A sample background task that divide two numbers."""
        try:
            # Enrich logs with task-specific data.
            log_context = {"task.input": {"x": x, "y": y}}
            celery_logger.info("Starting task execution.", extra=log_context)
            result = x / y
            celery_logger.info(f"Task finished", extra={"task.result": result})
            return result
        except Exception:
            celery_logger.error("Task failed!", exc_info=True)
            raise
 ```
 ### 2. Flask Application (Producer) (`flask_producer_app.py`)
 This file contains the Flask application. It's configured with deriqai-opentelemetry for both its Flask role and its Celery PRODUCER role. This is essential to ensure the trace context from an incoming web request is correctly passed to the Celery task.
 ```python
    # flask_producer_app.py
    import logging
    from flask import Flask
    from tasks import celery_app, my_background_task
    from deriqai import opentelemetry as dqotel

    def setup_observability(flask_app: Flask, celery_instance):
        """Configures OpenTelemetry for both Flask and the Celery producer."""
        resource = {"service.name": "flask-celery-producer", "service.version": "0.0.5", "deployment.environment": "production"}
        # 1. Configure for Flask to handle incoming web requests
        dqotel.configure(
            app=flask_app,
            app_type=dqotel.AppType.FLASK,
            resource=resource,
            level=logging.INFO
        )
        # 2. Configure for Celery Producer to propagate trace context
        dqotel.configure(
            app=celery_instance,
            app_type=dqotel.AppType.CELERY,
            role=dqotel.CeleryRole.PRODUCER,
            resource=resource,
            level=logging.INFO
        )

    app = Flask(__name__)
    setup_observability(app, celery_app)
    flask_logger = logging.getLogger("flask_app")

    @app.route('/')
    def index():
        """Endpoint to dispatch a background task."""
        flask_logger.info("Dispatching background task.")
        task = my_background_task.delay(10, 20)
        return f"Task {task.id} sent to worker."

    if __name__ == '__main__':
        app.run(debug=False, port=4000)
  ```
 ### 3. Celery Worker (`celery_worker.py`)
 This file is the entrypoint for the Celery worker. It configures deriqai-opentelemetry in its Celery WORKER role. This enables the worker to receive the trace context from the producer and continue the distributed trace.
 ```python
    # celery_worker.py
    import logging
    from deriqai import opentelemetry as dqotel
    from tasks import celery_app

    def setup_observability():
        """Configures OpenTelemetry for the Celery worker."""
        dqotel.configure(
            app=celery_app,
            app_type=dqotel.AppType.CELERY,
            role=dqotel.CeleryRole.WORKER,
            resource={"service.name": "my-celery-worker", "service.version": "0.2.3", "deployment.environment": "production"},
            level=logging.INFO
        )

    setup_observability()
    print("Celery worker instrumentation configured.")
 ```
 ### How to Run the Example
 **1. Install dependencies:** Make sure you have `celery`, `flask`, `deriqai-opentelemetry`, and a message broker (like RabbitMQ) installed and running.
 **2. Start the worker:** In one terminal, navigate to the directory containing the files and run:

```python
    celery -A celery_worker worker --loglevel=info
```

 **3. Start the Flask app:** In a second terminal, run:

```python
    python flask_producer_app.py
```

 **4. Trigger the task:** Open your browser and navigate to http://localhost:4000. You will see opentelmetry compatible logs confirming the task has been sent.

 **5. Observe the logs:** Check the terminal running the Celery worker. You will see logs indicating that the task was received and executed successfully. The logs will automatically include trace and span IDs, allowing you to follow the request through your observability tool.
