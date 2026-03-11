
Функция чтения XML в соответствие (удобно для анализ структуры)
Не учитывает схемы (про схемы смотреть [[XDTO]]). 

```bsl
function read(text) export
	xmlReader = new XMLReader;
	xmlReader.SetString(text);
	xmlReader.read();
	return convertToMap(xmlReader);
endfunction

function convertToMap(reader)
	
	if reader.NodeType = XMLNodeType.StartElement then
		tagName = reader.LocalName;
		args = new Map;
		content = new array;
		
		if reader.attributeCount() > 0 then
			while reader.ReadAttribute() do
				name = reader.name;
				value = reader.Value;
				args[name] = value;
			enddo;
		endif;
		
		// переход к первому дочернему узлу
		reader.read();
		
		while true do
			if reader.NodeType = XMLNodeType.EndElement then
				// конец текущего элемента
				break;
			elsif reader.NodeType = XMLNodeType.StartElement then
				content.add(convertToMap(reader));
				// после рекурсивного вызова reader уже на следующем узле, продолжаем цикл без дополнительного read
			elsif reader.NodeType = XMLNodeType.Text then
				content.add(reader.Value);
				reader.read(); // переходим к следующему узлу
			else
				raise "incorrect state";
			endif;
		enddo;
		
		// здесь reader на EndElement текущего элемента
		reader.read(); // переходим за него для родителя
		
		result = new map;
		result[tagName] = new structure("args, content", args, content);
		return result;
	elsif reader.NodeType = XMLNodeType.EndElement then
		reader.read();
	elsif reader.NodeType = XMLNodeType.Text then
		value = reader.Value;
		reader.read();
		return value;
	else
		raise strTemplate("reader.NodeType = '%1'", reader.NodeType);
	endif;
	
	
	
endfunction


```