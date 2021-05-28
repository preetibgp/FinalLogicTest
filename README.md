# FinalLogicTest


###Question
Make a simple todo app with flutter, just need two features in the app 
To add a todo
To display all todo
======>

import 'package:flutter/material.dart';

void main() {
  return runApp(MaterialApp(title: 'Todo', home: Tasks()));
}

/// Task model
/// A task contains a string text and a status completed.
class TaskModel extends ChangeNotifier {
  final String text;
  bool completed;

  TaskModel({this.text, this.completed = false});

  TaskModel.fromJson(Map<String, dynamic> json)
      : text = json['text'],
        completed = json['completed'];

  Map<String, dynamic> toJson() => {'text': text, 'completed': completed};

  void toggle() {
    completed = !completed;
    notifyListeners();
  }
}


class TodoModel extends ChangeNotifier {
  List<TaskModel> tasks = [];

  void addTaks(TaskModel task) {
    tasks.add(task);
    notifyListeners();
  }

  Future<void> getTasksFromSharedPrefs() async {
    final prefs = await SharedPreferences.getInstance();
    final tasksJson = prefs.getString('tasks') ?? '[]';
    // https://flutter.dev/docs/cookbook/networking/background-parsing#convert-the-response-into-a-list-of-photos
    final jsonListTasks = jsonDecode(tasksJson).cast<Map<String, dynamic>>();
    tasks = jsonListTasks.map<TaskModel>((m) => TaskModel.fromJson(m)).toList();
    notifyListeners();
  }

  Future<void> saveTasksToSharedPrefs() async {
    final prefs = await SharedPreferences.getInstance();
    final json = jsonEncode(tasks);
    prefs.setString('tasks', json);
  }

  List<TaskModel> getCompletedTasks() {
    return tasks.where((t) => t.completed == true).toList();
  }
}
  

class CompletedTasks extends StatelessWidget {
  final TodoListModel todoList;

  CompletedTasks({this.todoList});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          title: Text('Completed Items'),
        ),
        body: ListView(
            children: todoList.getCompletedTasks().map((t) {
          return Container(
            height: 50,
            child: Center(child: Text(t.text, style: TextStyle(fontSize: 20))));
        }).toList()));
  }
}
  
  class TaskWidget extends StatelessWidget {
  // method to style completed/uncompleted item
  TextStyle _taskStyle(completed) {
    if (completed)
      return TextStyle(
        color: Colors.black54,
        decoration: TextDecoration.lineThrough,
      );
    else
      return TextStyle(decoration: TextDecoration.none);
  }

  @override
  Widget build(BuildContext context) {
    return Consumer<TaskModel>(builder: (context, task, child) {
      return CheckboxListTile(
        title: Text(
          task.text,
          style: _taskStyle(task.completed),
        ),
        value: task.completed,
        onChanged: (newValue) {
          task.toggle();
          Provider.of<TodoListModel>(context, listen: false)
              .saveTasksToSharedPrefs();
        },
        controlAffinity: ListTileControlAffinity.leading,
      );
    });
  }
}
  
  class Tasks extends StatelessWidget {
  // display completed tasks screen
  void _goToCompletedTasks(context, todoList) {
    Navigator.push(
        context,
        MaterialPageRoute(
            builder: (context) => CompletedTasks(todoList: todoList)));
  }

  @override
  Widget build(BuildContext context) {
    // get tasks from shared preferences
    final TodoListModel todoList = TodoListModel();
    // getTasksFromSharedPrefs call notifyListeners
    todoList.getTasksFromSharedPrefs();

    return Scaffold(
        appBar: AppBar(
          title: Text('TodoList'),
          actions: [
            Padding(
                padding: EdgeInsets.only(right: 20.0),
                child: IconButton(
                    icon: Icon(Icons.check),
                    onPressed: () => _goToCompletedTasks(context, todoList)))
          ],
        ),
        body: ChangeNotifierProvider.value(
          value: todoList,
          child: TodoListWidget(),
        ));
  }
}
  
  class TodoListWidget extends StatelessWidget {
  final TextEditingController _controller = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Column(children: [
      Expanded(child: Consumer<TodoListModel>(builder: (context, tasks, child) {
        return ListView(
          children: tasks.tasks.map((TaskModel task) {
            return ChangeNotifierProvider.value(
                value: task, child: TaskWidget());
          }).toList(),
        );
      })),
      Consumer<TodoListModel>(
        builder: (context, tasks, child) {
          return TextField(
            controller: _controller,
            decoration: InputDecoration(
                border: OutlineInputBorder(
                    borderSide: BorderSide(color: Colors.teal)),
                labelText: 'new task'),
            onSubmitted: (newTask) {
              tasks.addTaks(TaskModel(
                  text:
                      newTask)); // create new instance of task changeNotifier model
              _controller.clear(); // clear the text input after adding taks
              tasks.saveTasksToSharedPrefs();
            },
          );
        },
      )
    ]);
  }
}
