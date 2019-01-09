### Calculate arithmetic expression

Calculate the result of giving arithmetic expression, with developed by Dijkstra in the 1960s uses two stacks ( one for operands and one for operators ), the rules as below:

* push operands onto the operand stack
* push operators onto the oeprator stack
* ignore left parentheses
* when meet a right parenthesis. pop an operator, pop the requisite number of operands, and push onto the operand stack the result of applying that operator to those operands

assume that always got valid expression ( also, subexpression surrounded by left and right parenthese ), ignore all error handling, an implementation could be:

```javascript

function calculate(expression) {
  const operands = [];
  const operators = [];
  
  for (let i = 0; i < expression.length; i++) {
        const v = expression.charAt(i);        
        if (v === ")") {
            const secondOperand = operands.pop();
            const firstOperand = operands.pop();
            
            operands.push(operators.pop()(firstOperand, secondOperand));            
        } else if (isOperator(v)) {
            operators.push((a, b) => {                
                if (v === "+") {
                    return a + b;
                } else if (v === "-") {
                    return a - b;
                } else if (v === "*") {
                    return a * b;
                } else if (v === "/") {
                    return a / b;
                }                
            });            
        } else if (parseInt(v) !== NaN) {
            operands.push(parseInt(v));            
        }            
    }
  
  return operands.pop();
}

function isOperator(operator) {
  return operator === "+" ||
  operator === "-" ||
  operator === "*" ||
  operator === "/"
}

```