Minimal Fake Standard Library
-----------------------------
visit_ImportFrom in [intermediateform.md](intermediateform.md) checks for the inline implementations here,
if not found, then the final backend or user needs to implement the import.


```python

class fakestdlib:
	REQUIRES = 1

	LUA = {
		'time': {
			## requires socket module, install for luajit on ubuntu - `sudo-apt get install lua-socket`
			## for lua interpreter on ubuntu - `sudo apt-get install liblua5.1-socket`
			REQUIRES : ['socket'],
			'time' : 'time = function() return socket.gettime() end',
			'clock' : 'clock = function() return socket.gettime() end'
		},
		'math': {
			'sin' : 'sin = function(a) return math.sin(a[1]) end',
			'cos' : 'cos = function(a) return math.cos(a[1]) end',
			'sqrt' : 'sqrt = function(a) return math.sqrt(a[1]) end',
		}
	}


	DART = {
		'time': {
			'time' : 'time() { return new DateTime.now().millisecondsSinceEpoch / 1000.0; }',
			'clock' : 'clock() { return new DateTime.now().millisecondsSinceEpoch / 1000.0; }'
		},
		'math': {
			'sin' : 'var sin = math.sin',
			'cos' : 'var cos = math.cos',
			'sqrt' : 'var sqrt = math.sqrt',
		},
		'random' : {
			'random' : 'var random = __random__'
		}

	}

	JS = {
		'time': {
			'time': 'function time() { return new Date().getTime() / 1000.0; }',
			'clock': 'function clock() { return new Date().getTime() / 1000.0; }'
		},
		'random': {
			'random': 'var random = Math.random'
		},
		'bisect' : {
			'bisect' : '/*bisect from fake bisect module*/'  ## bisect is a builtin
		},
		'math' : {
			'sin' : 'var sin = Math.sin',
			'cos' : 'var cos = Math.cos',
			'sqrt': 'var sqrt = Math.sqrt'
		},
		'os.path' : {
			'dirname' : "function dirname(s) { return s.slice(0, s.lastIndexOf('/')+1)}; var os = {'path':{'dirname':dirname}}"
		}
	}

	GO = {
		'time': {
			REQUIRES : ['time'],
			'clock': 'func clock() float64 { return float64(time.Now().Unix()); }'
		},
	}

	CPP = {
		'time': {
			'clock': 'double __clock__() { return std::chrono::duration_cast<std::chrono::nanoseconds>(std::chrono::high_resolution_clock::now().time_since_epoch()).count() / (double)1000000000.0; }',
			'sleep': 'void sleep(double seconds) { std::this_thread::sleep_for(std::chrono::milliseconds( (long)(1000.0*seconds) )); }',
		},

		'math': {
			'sin' : 'double sin(double a) { return std::sin(a); }',
			'cos' : 'double cos(double a) { return std::cos(a); }',
			'sqrt' : 'double sqrt(double a) { return std::sqrt(a); }',
		}
	}

```