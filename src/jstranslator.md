Javascript Translator
---------------------

translates intermediate form into final javascript.
This is also subclassed by these other backends:
* [gotranslator.md](gotranslator.md)
* [rusttranslator.md](rusttranslator.md)
* [cpptranslator.md](cpptranslator.md)


```python
# PythonJS to JavaScript Translator
# by Amirouche Boubekki and Brett Hartshorn - copyright 2013
# License: "New BSD"


from types import GeneratorType
from ast import Str
from ast import Name
from ast import Tuple
from ast import Attribute
from ast import NodeVisitor

class SwapLambda( RuntimeError ):
	def __init__(self, node):
		self.node = node
		RuntimeError.__init__(self)

class JSGenerator(NodeVisitorBase, GeneratorBase):
	def __init__(self, source, requirejs=True, insert_runtime=True, webworker=False, function_expressions=True, fast_javascript=False, fast_loops=False):
		assert source
		NodeVisitorBase.__init__(self, source)

		self._fast_js = fast_javascript
		self._fast_loops = fast_loops
		self._func_expressions = function_expressions
		self._indent = 0
		self._global_functions = {}
		self._function_stack = []
		self._requirejs = requirejs
		self._insert_runtime = insert_runtime
		self._webworker = webworker
		self._exports = set()
		self._inline_lambda = False
		self.catch_call = set()  ## subclasses can use this to catch special calls

		self.special_decorators = set(['__typedef__', '__glsl__', '__pyfunction__', 'expression'])
		self._glsl = False  ## TODO deprecate
		self._has_glsl = False  ## TODO deprecate
		self.glsl_runtime = 'int _imod(int a, int b) { return int(mod(float(a),float(b))); }'  ## TODO deprecate

		self._typed_vars = dict()

		self._lua  = False
		self._dart = False
		self._go   = False
		self._rust = False
		self._cpp = False
		self._cheader = []
		self._cppheader = []
		self._cpp_class_impl = []
		self._match_stack = []       ## dicts of cases
		self._rename_hacks = {}      ## used by c++ backend, to support `if isinstance`
		self._globals = {}           ## name : type
		self._called_functions = {}  ## name : number of calls
```

reset
-----
`reset()` needs to be called for multipass backends, that are dumb and run translation twice to gather info in two passes.

```python

	def reset(self):
		self._cheader = []
		self._cppheader = []
		self._cpp_class_impl = []
		self._match_stack = []

```

Class
------
class is not implemented here for javascript, it gets translated ahead of time in 
[intermediateform.md](intermediateform.md)


```python

	def visit_ClassDef(self, node):
		raise NotImplementedError(node)


	def visit_Global(self, node):
		return '/*globals: %s */' %','.join(node.names)

	def visit_Assign(self, node):
		target = node.targets[0]
		if isinstance(target, Tuple):
			raise NotImplementedError('target tuple assignment should have been transformed to flat assignment by python_to_pythonjs.py')
		else:
			target = self.visit(target)
			value = self.visit(node.value)
			code = '%s = %s;' % (target, value)
			if self._requirejs and target not in self._exports and self._indent == 0 and '.' not in target:
				self._exports.add( target )
			return code

	def visit_AugAssign(self, node):
		## n++ and n-- are slightly faster than n+=1 and n-=1
		target = self.visit(node.target)
		op = self.visit(node.op)
		value = self.visit(node.value)
		if op=='+' and isinstance(node.value, ast.Num) and node.value.n == 1:
			a = '%s ++;' %target
		if op=='-' and isinstance(node.value, ast.Num) and node.value.n == 1:
			a = '%s --;' %target
		else:
			a = '%s %s= %s;' %(target, op, value)
		return a

```

RequireJS
---------

generate a generic or requirejs module.

```python
	def _new_module(self, name='main.js'):
		header = []
		if self._requirejs and not self._webworker:
			header.extend([
				'define( function(){',
				'__module__ = {}'
			])

		return {
			'name'   : name,
			'header' : header,
			'lines'  : []
		}

```
Module
------
TODO: regenerate pythonjs.js each time.

```python

	def visit_Module(self, node):
		modules = []

		mod = self._new_module()
		modules.append( mod )
		lines = mod['lines']
		header = mod['header']


		for b in node.body:
			if isinstance(b, ast.Expr) and isinstance(b.value, ast.Call) and isinstance(b.value.func, ast.Name) and b.value.func.id == '__new_module__':
				mod = self._new_module( '%s.js' %b.value.args[0].id )
				modules.append( mod )
				lines = mod['lines']
				header = mod['header']

			else:
				line = self.visit(b)
				if line: lines.append( line )


		if self._insert_runtime:
			dirname = os.path.dirname(os.path.abspath(__file__))
			runtime = open( os.path.join(dirname, 'pythonjs/pythonjs.js') ).read()
			lines.insert( 0, runtime )


		if self._requirejs and not self._webworker:
			for name in self._exports:
				if name.startswith('__'): continue
				lines.append( '__module__.%s = %s' %(name,name))

			lines.append( 'return __module__')
			lines.append('}) //end requirejs define')


		if len(modules) == 1:
			lines = header + lines
			## fixed by Foxboron
			return '\n'.join(l if isinstance(l,str) else l.encode("utf-8") for l in lines)
		else:
			d = {}
			for mod in modules:
				lines = mod['header'] + mod['lines']
				d[ mod['name'] ] = '\n'.join(l if isinstance(l,str) else l.encode("utf-8") for l in lines)
			return d

```
In
----
note a `in` test in javascript is very different from the way python normally works,
for example `0 in [1,2,3]` is true in javascript, while it is false in python,
this is because an `in` test on an array in javascript checks the indices, not the values,
while in python it works by testing if the value is in the array.
Depending on the options given in the first stage of translation [intermediateform.md](intermediateform.md),
`in` tests will be replaced with a function call to `__contains__` which implements the python style logic.
However, in some cases an `in` test is still generated at here in the final stage of translation.

```python

	def visit_In(self, node):
		return ' in '

```
Try/Except and Raise
--------------
TODO `finnally` for the javascript backend 

```python

	def visit_TryExcept(self, node):
		out = []
		out.append( self.indent() + 'try {' )
		self.push()
		out.extend(
			list( map(self.visit, node.body) )
		)
		self.pull()
		out.append( self.indent() + '} catch(__exception__) {' )
		self.push()
		out.extend(
			list( map(self.visit, node.handlers) )
		)
		self.pull()
		out.append( '}' )
		return '\n'.join( out )

	def visit_Raise(self, node):
		if self._rust:
			return 'panic!("%s");'  % self.visit(node.type)
		elif self._cpp:
			T = self.visit(node.type)
			if T == 'RuntimeError()': T = 'std::exception'
			return 'throw %s;' % T
		else:
			return 'throw new %s;' % self.visit(node.type)

	def visit_ExceptHandler(self, node):
		out = ''
		if node.type:
			out = 'if (__exception__ == %s || __exception__ instanceof %s) {\n' % (self.visit(node.type), self.visit(node.type))
		if node.name:
			out += 'var %s = __exception__;\n' % self.visit(node.name)
		out += '\n'.join(map(self.visit, node.body)) + '\n'
		if node.type:
			out += '}\n'
		return out

```
Yield
------
note yield is new in javascript, and works slightly different from python, ie.
yielding is not cooperative, calling `some_function_that_also_yields()` inside
a function that is already using yield will not co-op yield.


```python

	def visit_Yield(self, node):
		return 'yield %s' % self.visit(node.value)

	def visit_Lambda(self, node):
		args = [self.visit(a) for a in node.args.args]
		if args and args[0]=='__INLINE_FUNCTION__':
			self._inline_lambda = True
			#return '<LambdaError>'   ## skip node, the next function contains the real def
			raise SwapLambda( node )
		else:
			return '(function (%s) {return %s;})' %(','.join(args), self.visit(node.body))


```

Function/Methods
----------------
note: `visit_Function` after doing some setup, calls `_visit_function` that subclasses overload.


```python

	def _visit_function(self, node):
		comments = []
		body = []
		is_main = node.name == 'main'
		is_annon = node.name == ''
		is_pyfunc    = False
		is_prototype = False
		protoname    = None
		func_expr    = False  ## function expressions `var a = function()` are not hoisted
		func_expr_var = True

		for decor in node.decorator_list:
			if isinstance(decor, ast.Call) and isinstance(decor.func, ast.Name) and decor.func.id == 'expression':
				assert len(decor.args)==1
				func_expr = True
				func_expr_var = isinstance(decor.args[0], ast.Name)
				node.name = self.visit(decor.args[0])

			elif isinstance(decor, ast.Name) and decor.id == '__pyfunction__':
				is_pyfunc = True
			elif isinstance(decor, ast.Call) and isinstance(decor.func, ast.Name) and decor.func.id == '__prototype__':  ## TODO deprecated
				assert len(decor.args)==1
				is_prototype = True
				protoname = decor.args[0].id

		args = self.visit(node.args)

		if is_prototype:
			fdef = '%s.prototype.%s = function(%s)' % (protoname, node.name, ', '.join(args))

		elif len(self._function_stack) == 1:
			## this style will not make function global to the eval context in NodeJS ##
			#buffer = self.indent() + 'function %s(%s) {\n' % (node.name, ', '.join(args))

			## note if there is no var keyword and this function is at the global level,
			## then it should be callable from eval in NodeJS - this is not correct.
			## infact, var should always be used with function expressions.

			if self._func_expressions or func_expr:
				if func_expr_var:
					fdef = 'var %s = function(%s)' % (node.name, ', '.join(args))
				else:
					fdef = '%s = function(%s)' % (node.name, ', '.join(args))
			else:
				fdef = 'function %s(%s)' % (node.name, ', '.join(args))


			if self._requirejs and node.name not in self._exports:
				self._exports.add( node.name )

		else:

			if self._func_expressions or func_expr:
				if func_expr_var:
					fdef = 'var %s = function(%s)' % (node.name, ', '.join(args))
				else:
					fdef = '%s = function(%s)' % (node.name, ', '.join(args))
			else:
				fdef = 'function %s(%s)' % (node.name, ', '.join(args))

		body.append( fdef )

		body.append( self.indent() + '{' )
		self.push()
		next = None
		for i,child in enumerate(node.body):
			if isinstance(child, Str) or hasattr(child, 'SKIP'):
				continue
			elif isinstance(child, ast.Expr) and isinstance(child.value, ast.Str):
				comments.append('/* %s */' %child.value.s.strip() )
				continue

			#try:
			#	v = self.visit(child)
			#except SwapLambda as error:
			#	error.node.__class__ = ast.FunctionDef
			#	next = node.body[i+1]
			#	if not isinstance(next, ast.FunctionDef):
			#		raise SyntaxError('inline def is only allowed in javascript mode')
			#	error.node.__dict__ = next.__dict__
			#	error.node.name = ''
			#	v = self.visit(child)

			v = self.try_and_catch_swap_lambda(child, node.body)


			if v is None:
				msg = 'error in function: %s'%node.name
				msg += '\n%s' %child
				raise SyntaxError(msg)
			else:
				body.append( self.indent()+v)

		#buffer += '\n'.join(body)
		self.pull()
		#buffer += '\n%s}' %self.indent()

		body.append( self.indent() + '}' )

		buffer = '\n'.join( comments + body )

		#if self._inline_lambda:
		#	self._inline_lambda = False
		if is_annon:
			buffer = '__wrap_function__(' + buffer + ')'
		elif is_pyfunc:
			## TODO change .is_wrapper to .__pyfunc__
			buffer += ';%s.is_wrapper = true;' %node.name
		else:
			buffer += '\n'

		return self.indent() + buffer

	def try_and_catch_swap_lambda(self, child, body):
		try:
			return self.visit(child)
		except SwapLambda as e:

			next = None
			for i in range( body.index(child), len(body) ):
				n = body[ i ]
				if isinstance(n, ast.FunctionDef):
					if hasattr(n, 'SKIP'):
						continue
					else:
						next = n
						break
			assert next
			next.SKIP = True
			e.node.__class__ = ast.FunctionDef
			e.node.__dict__ = next.__dict__
			e.node.name = ''
			return self.try_and_catch_swap_lambda( child, body )


	def _visit_subscript_ellipsis(self, node):
		name = self.visit(node.value)
		return '%s["$wrapped"]' %name


	def visit_Slice(self, node):
		raise SyntaxError('list slice')  ## slicing not allowed here at js level

	def visit_arguments(self, node):
		out = []
		for name in [self.visit(arg) for arg in node.args]:
			out.append(name)
		return out

	def visit_Name(self, node):
		if node.id == 'None':
			return 'null'
		elif node.id == 'True':
			return 'true'
		elif node.id == 'False':
			return 'false'
		elif node.id == 'null':
			return 'null'
		elif node.id == '__DOLLAR__':
			return '$'
		return node.id

	def visit_Attribute(self, node):
		name = self.visit(node.value)
		attr = node.attr
		return '%s.%s' % (name, attr)

	def visit_Print(self, node):
		args = [self.visit(e) for e in node.values]
		s = 'console.log(%s);' % ', '.join(args)
		return s

	def visit_keyword(self, node):
		if isinstance(node.arg, basestring):
			return node.arg, self.visit(node.value)
		return self.visit(node.arg), self.visit(node.value)

```

Call Helper
------------


```python

	def _visit_call_helper(self, node):
		if node.args:
			args = [self.visit(e) for e in node.args]
			args = ', '.join([e for e in args if e])
		else:
			args = ''
		fname = self.visit(node.func)
		if fname=='__DOLLAR__': fname = '$'
		return '%s(%s)' % (fname, args)

	def inline_helper_remap_names(self, remap):
		return "var %s;" %','.join(remap.values())

	def inline_helper_return_id(self, return_id):
		return "var __returns__%s = null;"%return_id

	def _visit_call_helper_numpy_array(self, node):
		return self.visit(node.args[0])

	def _visit_call_helper_list(self, node):
		name = self.visit(node.func)
		if node.args:
			args = [self.visit(e) for e in node.args]
			args = ', '.join([e for e in args if e])
		else:
			args = ''
		return '%s(%s)' % (name, args)

	def _visit_call_helper_get_call_special(self, node):
		name = self.visit(node.func)
		if node.args:
			args = [self.visit(e) for e in node.args]
			args = ', '.join([e for e in args if e])
		else:
			args = ''
		return '%s(%s)' % (name, args)


	def _visit_call_helper_JSArray(self, node):
		if node.args:
			args = map(self.visit, node.args)
			out = ', '.join(args)
			#return '__create_array__(%s)' % out
			return '[%s]' % out

		else:
			return '[]'


	def _visit_call_helper_JSObject(self, node):
		if node.keywords:
			kwargs = map(self.visit, node.keywords)
			f = lambda x: '"%s": %s' % (x[0], x[1])
			out = ', '.join(map(f, kwargs))
			return '{%s}' % out
		else:
			return '{}'

	def _visit_call_helper_var(self, node):
		args = [ self.visit(a) for a in node.args ]
		if self._function_stack:
			fnode = self._function_stack[-1]
			rem = []
			for arg in args:
				if arg in fnode._local_vars:
					rem.append( arg )
				else:
					fnode._local_vars.add( arg )
			for arg in rem:
				args.remove( arg )
		out = []
		if args:
			out.append( 'var ' + ','.join(args) )
		if node.keywords:
			out.append( 'var ' + ','.join([key.arg for key in node.keywords]) )
		return ';'.join(out)

```

Inline Code Helper
------------------

TODO clean this up

```python

	def _inline_code_helper(self, s):
		## TODO, should newline be changed here?
		s = s.replace('\n', '\\n').replace('\0', '\\0')  ## AttributeError: 'BinOp' object has no attribute 's' - this is caused by bad quotes
		if s.strip().startswith('#'): s = '/*%s*/'%s

		if '__new__>>' in s:
			## hack that fixes inline `JS("new XXX")`,
			## TODO improve typedpython to be aware of quoted strings
			s = s.replace('__new__>>', ' new ')

		elif '"' in s or "'" in s:  ## can not trust direct-replace hacks
			pass
		else:
			if ' or ' in s:
				s = s.replace(' or ', ' || ')
			if ' not ' in s:
				s = s.replace(' not ', ' ! ')
			if ' and ' in s:
				s = s.replace(' and ', ' && ')
		return s

	def visit_While(self, node):
		body = [ 'while (%s)' %self.visit(node.test), self.indent()+'{']
		self.push()
		for line in list( map(self.visit, node.body) ):
			body.append( self.indent()+line )
		self.pull()
		body.append( self.indent() + '}' )
		return '\n'.join( body )

	def visit_Str(self, node):
		s = node.s.replace("\\", "\\\\").replace('\n', '\\n').replace('\r', '\\r').replace('"', '\\"')
		#if '"' in s:
		#	return "'%s'" % s
		return '"%s"' % s

	def visit_BinOp(self, node):
		left = self.visit(node.left)
		op = self.visit(node.op)
		right = self.visit(node.right)
		go_hacks = ('__go__array__', '__go__arrayfixed__', '__go__map__', '__go__func__', '__go__receive__', '__go__send__')

		if op == '>>' and left == '__new__':
			## this can happen because python_to_pythonjs.py will catch when a new class instance is created
			## (when it knows that class name) and replace it with `new(MyClass())`; but this can cause a problem
			## if later the user changes that part of their code into a module, and loads it as a javascript module,
			## they may update their code to call `new MyClass`, and then later go back to the python library.
			## the following hack prevents `new new`
			if isinstance(node.right, ast.Call) and isinstance(node.right.func, ast.Name) and node.right.func.id=='new':
				right = self.visit(node.right.args[0])
			return ' new %s' %right


		elif op == '<<':

			if left in ('__go__receive__', '__go__send__'):
				self._has_channels = True
				return '%s.recv()' %right

			if isinstance(node.left, ast.Call) and isinstance(node.left.func, ast.Name) and node.left.func.id in go_hacks:
				if node.left.func.id == '__go__func__':
					raise SyntaxError('TODO - go.func')
				elif node.left.func.id == '__go__map__':
					key_type = self.visit(node.left.args[0])
					value_type = self.visit(node.left.args[1])
					if value_type == 'interface': value_type = 'interface{}'
					return '&map[%s]%s%s' %(key_type, value_type, right)
				else:
					if isinstance(node.right, ast.Name):
						raise SyntaxError(node.right.id)

					right = []
					for elt in node.right.elts:
						if isinstance(elt, ast.Num):
							right.append( str(elt.n)+'i' )
						else:
							right.append( self.visit(elt) )
					right = '(%s)' %','.join(right)

					if node.left.func.id == '__go__array__':
						T = self.visit(node.left.args[0])
						if T in go_types:
							#return '&mut vec!%s' %right
							return 'Rc::new(RefCell::new(vec!%s))' %right
						else:
							self._catch_assignment = {'class':T}  ## visit_Assign catches this
							return '&[]*%s%s' %(T, right)

					elif node.left.func.id == '__go__arrayfixed__':
						asize = self.visit(node.left.args[0])
						atype = self.visit(node.left.args[1])
						return ' new Array(%s) /*array of: %s*/' %(asize, atype)

		if left in self._typed_vars and self._typed_vars[left] == 'numpy.float32':
			left += '[_id_]'
		if right in self._typed_vars and self._typed_vars[right] == 'numpy.float32':
			right += '[_id_]'

		return '(%s %s %s)' % (left, op, right)


	def visit_Return(self, node):
		if isinstance(node.value, Tuple):
			return 'return [%s];' % ', '.join(map(self.visit, node.value.elts))
		if node.value:
			return 'return %s;' % self.visit(node.value)
		return 'return undefined;'

	def visit_Pass(self, node):
		return '/*pass*/'

	def visit_Is(self, node):
		return '==='

	def visit_IsNot(self, node):
		return '!=='


	def visit_Compare(self, node):
		if isinstance(node.ops[0], ast.Eq):
			left = self.visit(node.left)
			right = self.visit(node.comparators[0])
			if self._lua:
				return '%s == %s' %(left, right)
			elif self._fast_js:
				return '(%s===%s)' %(left, right)
			else:
				return '(%s instanceof Array ? JSON.stringify(%s)==JSON.stringify(%s) : %s===%s)' %(left, left, right, left, right)
		elif isinstance(node.ops[0], ast.NotEq):
			left = self.visit(node.left)
			right = self.visit(node.comparators[0])
			if self._lua:
				return '%s ~= %s' %(left, right)
			elif self._fast_js:
				return '(%s!==%s)' %(left, right)
			else:
				return '(!(%s instanceof Array ? JSON.stringify(%s)==JSON.stringify(%s) : %s===%s))' %(left, left, right, left, right)

		else:
			comp = [ '(']
			comp.append( self.visit(node.left) )
			comp.append( ')' )

			for i in range( len(node.ops) ):
				comp.append( self.visit(node.ops[i]) )

				if isinstance(node.ops[i], ast.Eq):
					raise SyntaxError('TODO')

				elif isinstance(node.comparators[i], ast.BinOp):
					comp.append('(')
					comp.append( self.visit(node.comparators[i]) )
					comp.append(')')
				else:
					comp.append( self.visit(node.comparators[i]) )

			return ' '.join( comp )


	def visit_UnaryOp(self, node):
		#return self.visit(node.op) + self.visit(node.operand)
		return '%s (%s)' %(self.visit(node.op),self.visit(node.operand))


	def visit_BoolOp(self, node):
		op = self.visit(node.op)
		return '('+ op.join( [self.visit(v) for v in node.values] ) +')'

```
If Test
-------


```python

	def visit_If(self, node):
		out = []
		test = self.visit(node.test)
		if test.startswith('(') and test.endswith(')'):
			out.append( 'if %s' %test )
		else:
			out.append( 'if (%s)' %test )
		out.append( self.indent() + '{' )

		self.push()

		for line in list(map(self.visit, node.body)):
			if line is None: continue
			out.append( self.indent() + line )

		orelse = []
		for line in list(map(self.visit, node.orelse)):
			orelse.append( self.indent() + line )

		self.pull()

		if orelse:
			out.append( self.indent() + '}')
			out.append( self.indent() + 'else')
			out.append( self.indent() + '{')
			out.extend( orelse )

		out.append( self.indent() + '}' )

		return '\n'.join( out )


	def visit_Dict(self, node):
		a = []
		for i in range( len(node.keys) ):
			k = self.visit( node.keys[ i ] )
			v = self.visit( node.values[i] )
			a.append( '%s:%s'%(k,v) )
		b = ', '.join( a )
		return '{ %s }' %b

```
For Loop
--------
when fast_loops is off much of python `for in something` style of looping is lost.


```python

	def _visit_for_prep_iter_helper(self, node, out, iter_name):
		## support "for key in JSObject" ##
		#out.append( self.indent() + 'if (! (iter instanceof Array) ) { iter = Object.keys(iter) }' )
		## new style - Object.keys only works for normal JS-objects, not ones created with `Object.create(null)`
		if not self._fast_loops:
			out.append(
				self.indent() + 'if (! (%s instanceof Array || typeof %s == "string" || __is_typed_array(%s) || __is_some_array(%s) )) { %s = __object_keys__(%s) }' %(iter_name, iter_name, iter_name, iter_name, iter_name, iter_name)
			)


	_iter_id = 0
	def visit_For(self, node):

		target = node.target.id
		iter = self.visit(node.iter) # iter is the python iterator

		out = []
		body = []

		self._iter_id += 1
		index = '__i%s' %self._iter_id
		if not self._fast_loops:
			iname = '__iter%s' %self._iter_id
			out.append( self.indent() + 'var %s = %s;' % (iname, iter) )
		else:
			iname = iter

		self._visit_for_prep_iter_helper(node, out, iname)

		if self._fast_loops:
			out.append( 'for (var %s=0; %s < %s.length; %s++)' % (index, index, iname, index) )
			out.append( self.indent() + '{' )

		else:
			out.append( self.indent() + 'for (var %s=0; %s < %s.length; %s++) {' % (index, index, iname, index) )
		self.push()

		body.append( self.indent() + 'var %s = %s[ %s ];' %(target, iname, index) )

		for line in list(map(self.visit, node.body)):
			body.append( self.indent() + line )

		self.pull()
		out.extend( body )
		out.append( self.indent() + '}' )

		return '\n'.join( out )

	def visit_Continue(self, node):
		return 'continue'

	def visit_Break(self, node):
		return 'break;'

```

Regenerate JS Runtime
---------------------

TODO: update and test generate new js runtimes

```python
def generate_minimal_js_runtime():
	from python_to_pythonjs import main as py2pyjs
	a = py2pyjs(
		open('src/runtime/builtins_core.py', 'rb').read(),
		module_path = 'runtime',
		fast_javascript = True
	)
	return main( a, requirejs=False, insert_runtime=False, function_expressions=True, fast_javascript=True )

def generate_js_runtime():
	from python_to_pythonjs import main as py2pyjs
	builtins = py2pyjs(
		open('src/runtime/builtins.py', 'rb').read(),
		module_path = 'runtime',
		fast_javascript = True
	)
	lines = [
		main( open('src/runtime/pythonpythonjs.py', 'rb').read(), requirejs=False, insert_runtime=False, function_expressions=True, fast_javascript=True ), ## lowlevel pythonjs
		main( builtins, requirejs=False, insert_runtime=False, function_expressions=True, fast_javascript=True )
	]
	return '\n'.join( lines )

```

Translate to Javascript
-----------------------
html files can also be translated, it is parsed and checked for `<script type="text/python">`

```python

def translate_to_javascript(source, requirejs=True, insert_runtime=True, webworker=False, function_expressions=True, fast_javascript=False, fast_loops=False):
	head = []
	tail = []
	script = False
	osource = source
	if source.strip().startswith('<html'):
		lines = source.splitlines()
		for line in lines:
			if line.strip().startswith('<script') and 'type="text/python"' in line:
				head.append( '<script type="text/javascript">')
				script = list()
			elif line.strip() == '</script>':
				if type(script) is list:
					source = '\n'.join(script)
					script = True
					tail.append( '</script>')
				elif script is True:
					tail.append( '</script>')
				else:
					head.append( '</script>')

			elif isinstance( script, list ):
				script.append( line )

			elif script is True:
				tail.append( line )

			else:
				head.append( line )


	try:
		tree = ast.parse( source )
		#raise SyntaxError(source)
	except SyntaxError:
		import traceback
		err = traceback.format_exc()
		sys.stderr.write( err )
		sys.stderr.write( '\n--------------error in second stage translation--------------\n' )

		lineno = 0
		for line in err.splitlines():
			if "<unknown>" in line:
				lineno = int(line.split()[-1])


		lines = source.splitlines()
		if lineno > 10:
			for i in range(lineno-5, lineno+5):
				sys.stderr.write( 'line %s->'%i )
				sys.stderr.write( lines[i] )
				if i==lineno-1:
					sys.stderr.write('  <<SyntaxError>>')
				sys.stderr.write( '\n' )

		else:
			sys.stderr.write( lines[lineno] )
			sys.stderr.write( '\n' )

		if '--debug' in sys.argv:
			sys.stderr.write( osource )
			sys.stderr.write( '\n' )

		sys.exit(1)

	gen = JSGenerator(
		source = source,
		requirejs=requirejs, 
		insert_runtime=insert_runtime, 
		webworker=webworker, 
		function_expressions=function_expressions,
		fast_javascript = fast_javascript,
		fast_loops      = fast_loops
	)
	output = gen.visit(tree)

	if head and not isinstance(output, dict):
		head.append( output )
		head.extend( tail )
		output = '\n'.join( head )

	return output


```