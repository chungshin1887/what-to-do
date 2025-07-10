from flask import Flask, request, redirect, url_for, render_template_string, jsonify

app = Flask(__name__)

todo_list = []  # {'task': 할 일 문자열, 'done': 완료 여부}

html = '''
<!doctype html>
<html>
<head>
  <title>To-Do 리스트</title>
  <style>
    body {
      font-family: '맑은 고딕', AppleSDGothicNeo, sans-serif;
      background: #000;
      color: white;
      display: flex;
      justify-content: center;
      align-items: flex-start;
      height: 100vh;
      margin: 0;
      padding-top: 50px;
    }
    .container {
      background: #111;
      max-width: 600px;
      width: 100%;
      padding: 20px;
      box-shadow: 0 0 10px rgba(255,255,255,0.1);
      border-radius: 8px;
    }
    h1 {
      text-align: center;
      color: #fff;
    }
    form.add-form {
      display: flex;
      margin-bottom: 20px;
    }
    form.add-form input[type="text"] {
      flex-grow: 1;
      padding: 10px;
      font-size: 16px;
      border: 1px solid #555;
      border-radius: 4px 0 0 4px;
      outline: none;
      background: transparent;
      color: white;
    }
    form.add-form input[type="submit"] {
      padding: 10px 20px;
      font-size: 16px;
      border: none;
      background: rgba(255, 255, 255, 0.3);
      color: white;
      cursor: pointer;
      border-radius: 0 4px 4px 0;
      transition: background 0.3s;
    }
    form.add-form input[type="submit"]:hover {
      background: #333333;
    }
    ul {
      list-style: none;
      padding: 0;
      user-select: none;
    }
    li {
      background: #222;
      margin-bottom: 10px;
      padding: 10px 15px;
      border-radius: 4px;
      box-shadow: 0 1px 3px rgba(255,255,255,0.1);
      display: flex;
      align-items: center;
      justify-content: space-between;
      cursor: grab;
      position: relative;
    }
    li.done span.task-text {
      text-decoration: line-through;
      color: gray;
    }
    span.task-text {
      flex-grow: 1;
      margin-left: 10px;
      user-select: text;
    }
    .buttons {
      display: none;
    }
    li.editing .buttons {
      display: inline;
    }
    .buttons form {
      display: inline;
      margin-left: 5px;
    }
    button, input[type="submit"] {
      background: rgba(255, 255, 255, 0.3);
      border: none;
      color: white;
      padding: 5px 10px;
      border-radius: 3px;
      cursor: pointer;
      font-size: 14px;
      transition: background 0.3s;
    }
    button:hover, input[type="submit"]:hover {
      background: #333333;
    }
    input[type="text"].edit-input {
      font-size: 16px;
      padding: 5px;
      width: 100%;
      border: 1px solid #555;
      border-radius: 4px;
      background: #111;
      color: white;
    }
    input[type="checkbox"] {
      width: 18px;
      height: 18px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>To-Do 리스트</h1>
    <form action="/add" method="post" class="add-form">
      <input type="text" name="task" placeholder="할 일을 입력하세요" required autocomplete="off">
      <input type="submit" value="추가">
    </form>

    <ul id="todo-list">
      {% for i, item in enumerate(tasks) %}
        <li class="{{ 'done' if item.done else '' }}" data-index="{{ i }}">
          <form action="/toggle/{{ i }}" method="post" style="margin:0;">
            <input type="checkbox" name="done" onchange="this.form.submit()" {% if item.done %}checked{% endif %}>
          </form>
          <span class="task-text">{{ item.task }}</span>
          <div class="buttons">
            <form action="/delete/{{ i }}" method="post" style="display:inline;">
              <button type="submit" onclick="return confirm('정말 삭제할까요?')">삭제</button>
            </form>
          </div>
        </li>
      {% endfor %}
    </ul>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.0/Sortable.min.js"></script>
  <script>
    window.onload = function() {
      const list = document.getElementById('todo-list');

      Sortable.create(list, {
        animation: 150,
        onEnd: function (evt) {
          const order = [];
          list.querySelectorAll('li').forEach(li => {
            order.push(li.dataset.index);
          });

          fetch('/reorder', {
            method: 'POST',
            headers: {'Content-Type':'application/json'},
            body: JSON.stringify({order: order})
          });
        }
      });

      list.addEventListener('dblclick', e => {
        if(e.target.classList.contains('task-text')) {
          const span = e.target;
          const currentText = span.textContent;
          const li = span.closest('li');
          const index = li.dataset.index;

          li.classList.add('editing');

          const input = document.createElement('input');
          input.type = 'text';
          input.value = currentText;
          input.className = 'edit-input';
          span.replaceWith(input);
          input.focus();

          input.addEventListener('blur', () => {
            fetch('/edit/' + index, {
              method: 'POST',
              headers: {'Content-Type':'application/x-www-form-urlencoded'},
              body: 'task=' + encodeURIComponent(input.value)
            }).then(() => {
              span.textContent = input.value;
              input.replaceWith(span);
              li.classList.remove('editing');
            });
          });

          input.addEventListener('keydown', (event) => {
            if(event.key === 'Enter'){
              input.blur();
            }
          });
        }
      });
    }
  </script>
</body>
</html>
'''

@app.route('/')
def index():
    return render_template_string(html, tasks=todo_list, enumerate=enumerate)

@app.route('/add', methods=['POST'])
def add():
    task = request.form['task'].strip()
    if task:
        todo_list.append({'task': task, 'done': False})
    return redirect(url_for('index'))

@app.route('/delete/<int:index>', methods=['POST'])
def delete(index):
    if 0 <= index < len(todo_list):
        todo_list.pop(index)
    return redirect(url_for('index'))

@app.route('/edit/<int:index>', methods=['POST'])
def edit(index):
    if 0 <= index < len(todo_list):
        task = request.form['task'].strip()
        if task:
            todo_list[index]['task'] = task
    return ('', 204)

@app.route('/reorder', methods=['POST'])
def reorder():
    order = request.json.get('order', [])
    global todo_list
    try:
        new_list = [todo_list[int(i)] for i in order]
        todo_list = new_list
        return jsonify({'result': 'success'})
    except Exception:
        return jsonify({'result': 'error'}), 400

@app.route('/toggle/<int:index>', methods=['POST'])
def toggle(index):
    if 0 <= index < len(todo_list):
        todo_list[index]['done'] = not todo_list[index]['done']
    return redirect(url_for('index'))

if __name__ == '__main__':
    app.run(debug=True)
