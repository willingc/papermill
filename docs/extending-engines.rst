Extending papermill - Custom Engines
====================================

A papermill engine is a python object that can run, or execute, a notebook. The
default implementation in papermill for example takes in a notebook object, and
runs it locally on your machine.

By writing a custom engine, you could allow execution to be handled remotely, or
you could apply post-processing to the executed notebook. In the next section,
you will see a demonstration.

Creating a new engine
---------------------

Papermill engines need to inherit from the ``papermill.engines.Engine`` class.

In order to be used, the new class needs to implement the class method
``execute_managed_notebook``. The call signature should match that of the parent
class:

.. code-block:: python

    class CustomEngine(papermill.engines.Engine):

        @classmethod
        execute_managed_notebook(cls, nb_man, kernel_name, **kwargs):
            pass

``nb_man`` is a |nbformat.NotebookNode|_, and ``kernel_name`` is a string. Your
custom class then needs to implement the execution of the notebook. For example,
you could insert code that executes the notebook remotely on a server, or
executes the notebook many times to simulate different conditions.

As an example, the following project implements a custom engine that adds the
time it took to execute each cell as additional output after every code cell.

The project structure is::

    papermill_timing
        |- setup.py
        |- src
            |- papermill_timing
                |- __init__.py


The file ``src/papermill_timing/__init__.py`` will implement the engine. Since
papermill already stores information about execution timing in the metadata,
we can leverage the default engine. We will also need to use the ``nbformat``
library to create a `notebook node object`_.

.. code-block:: python

    from datetime import datetime
    from papermill.engines import NBConvertEngine
    from nbformat.v4 import new_output

    class CustomEngine(NBConvertEngine):

        @classmethod
        def execute_managed_notebook(cls, nb_man, kernel_name, **kwargs):

            # call the papermill execution engine:
            super().execute_managed_notebook(nb_man, kernel_name, **kwargs)

            for cell in nb_man.nb.cells:

                if cell.cell_type == "code" and cell.execution_count is not None:
                    start = datetime.fromisoformat(cell.metadata.papermill.start_time)
                    end = datetime.fromisoformat(cell.metadata.papermill.end_time)
                    output_message = f"Execution took {(end - start).total_seconds():.3f} seconds"
                    output_node = new_output("display_data", data={"text/plain": [output_message]})
                    cell.outputs = [output_node] + cell.outputs

Once this is in place, we need to add our engine as an entry point to our
``setup.py`` script - for this, see the following section.

Ensuring your engine is found by papermill
------------------------------------------

Custom engines can be specified as `entry points`_, under the
``papermill.engine`` prefix. The entry point needs to reference the class that
we have just implemented. For example, if you write an engine called
TimingEngine in a package called papermill_timing, then in the ``setup.py``
file, you should specify:

.. code-block:: python

    from setuptools import setup, find_packages

    setup(
        name="papermill_timing",
        version="0.1",
        url="https://github.com/my_username/papermill_timing.git",
        author="My Name",
        author_email="my.email@gmail.com",
        description="A papermill engine that logs additional timing information about code.",
        packages=find_packages("./src"),
        package_dir={"": "src"},
        install_requires=["papermill", "nbformat"],
        entry_points={"papermill.engine": ["timer_engine=papermill_timing:TimingEngine"]},
    )

This allows users to specify the engine from ``papermill_timing`` by passing the
command line argument ``--engine timer_engine``.

In the image below, the notebook on the left was executed with the new custom
engine, while the one on the left was executed with the standard papermill
engine. As you can see, this adds our "injected" output to each code cell

.. image:: img/custom_execution_engine.png

.. |nbformat.NotebookNode| replace:: ``nbformat.NotebookNode`` object
.. _nbformat.NotebookNode: https://nbformat.readthedocs.io/en/latest/api.html#notebooknode-objects
.. _`notebook node object`: https://nbformat.readthedocs.io/en/latest/api.html#module-nbformat.v4