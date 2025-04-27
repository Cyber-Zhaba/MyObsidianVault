---
tags:
  - "#task"
deadline: <% tp.date.now("YYYY-MM-DDTHH:mm") %>
priority: <%* 
  const priorities = [
    {name: "🔴 Максимальный", desc: "Трубы горят! Сделать в первую очередь"},
    {name: "🟠 Высокий", desc: "Важно, но можно немного подождать"},
    {name: "🟡 Средний", desc: "Обычный приоритет"},
    {name: "🟢 Низкий", desc: "Когда будет время"}
  ];
  const choice = await tp.system.suggester(
    p => `${p.name} - ${p.desc}`,
    priorities
  );
  tR += `${choice.name}`;
%>
status: 📁 Не начато
completed: false
---
