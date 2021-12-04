# Neural Nets in Rkua (Part 1)

Thinky the Elf was sitting in his office, it had been a closet but he'd been given it as his office after the great baked beans incident. It wasn't his fault. He was right that feeding the reindeer beans would give them *a jet boost* but Santa had not been all that happy about it. And his tendency to stare of into space while suddenly having a thought wasn't great while working on the shop floor meant it was safer to put him out of the way to do some thinking.

Recently he'd been thinking about how to sort children into naughty or nice. This was Santa's big job all year and Thinky thought that there must be a way to simplify it, he'd spent some time watching videos on YouTube and there was one that gave a (brilliant description of Neural Networks)[https://www.youtube.com/watch?v=N3tRFayqVtk] (jump to 20 minutes for that bit but it's an interesting video). As Thinky whatched this he couldn't help thinking about Raku and how the connections between nodes felt like Supplies.

With this he dived in and played about to try and build Neural Networks with Raku and Supplies, he tried a few things and got to a system that worked but it had a few drawbacks.

Firstly we start with a Neuron role. The Neuron might be and input, and intermediate grouping one or a final output one but they have some shared functionality.

```
role Neuron {
    has %!input-vals = {};
    has Supply $!die;
    has Supply $!input;
    has Str $.id is required;
    has Str $.gene;
    has Bool $.scream;

    submethod BUILD( :$!die, :$!id, :$!gene = '', :$!scream = False ) {}

```

`%input-vals` stores the inputs this Neuron has recieved. The `$!die` Supply will recieve a message when the Neuron (and it's contiaining Brain) is to stop whilst the `$!input` Supply takes in all the input data. Each Neuron has a unique `id` and also knows the `gene` used to create the Brain it is found in. The `scream` Boolean will cause it to emit tracking info via notes.

Then there are a few methods :

```
    method !process-inputs() {...}
```

`process-inputs` is a placeholder for the concrete Neuron classes to handle what to do with incoming data.

```
    method start() {
        my $alive = True;
        if ( ! $!input.defined ) {
            return;
        }
        return start react {
            whenever $!die -> $ {
                note "{$!gene} : {$!id} : DIE" if $!scream;
                $alive = False;
                done();
            }
            whenever $!input -> ($id, $v) {
                note "{$!gene} : {$!id} : {$id} : {$v} : {$alive}" if $!scream;
                %!input-vals{$id} = $v;
                self!process-inputs();
                done() unless $alive;
            }
        }
    }
```

`start` brings a Neuron to live and returns a Promise (or if the Neuron isn't wired up with inputs nothing). The Neuron watches it's two supplies and fires off `process-inputs` after updating the `%!input-vals` hash. If it receives a trigger on the `$!die` supply it shuts itself down and tells any other processes will running to do so too by setting the internal `alive` Boolean. 

```
    method attach-input(Supply $s) {
        if ( ! $!input ) {
            $!input = Supply.merge( $s );
        } else {
            $!input = $!input.merge($s);
        }
    }
}
```

`attach-input` uses the `merge` method to combine all the inputs passed into a Neuron into one single one that the `start` method watches.

Two of the Neurons, the Input and Group, can can have multiple outputs so we'll make a Role for them.

```
role PassThruNeuron does Neuron {
    has @.outputs;

    method attach-output( Supplier $out ) {
        @!outputs.push( $out );
    }
}
```

Then we defined the Input and Group Neurons as PassThrus.

```
class InputNeuron does PassThruNeuron {
    method !process-inputs() {
        if ( %!input-vals{$!id}.defined ) {
            .emit( ( $!id, %!input-vals{$!id} ) ) for @!outputs;
        }
    }
}
```

The Input neuron filters any input data it's recieved for it's id only and send's this onto it's outputs. This allows us to have one shared Input stream that inputs can pull from. 

```
class TanHGroupNeuron does PassThruNeuron {
    has Rat $!threshold = 0.1;
    has $!previous;

    method !process-inputs() {
        .emit( ( $!id, tanh( [+] %!input-vals.values ).round($!threshold) ) ) for @!outputs;
    }
}
```

The TanHGroupNeuron (called as such to allow for mulitple type of Group Neurons) computes the tanh value of the sum of it's inputs and sends these out.

And then we have the Output Neuron, it's only got one output value so it's pretty simple.

```
class OutputNeuron does Neuron {
    has Num $!output;
    has Rat $!threshold = 0.1;

    method !process-inputs() {
        $!output = tanh( [+] %!input-vals.values );
    }

    method output() {
        $!output.defined ?? $!output.round($!threshold) !! 0;
    }
}
```

Once again we're rounding the value output and if there isn't a value set we return a 0. Note that we have `$!threshold` values to manage rounding. These are currently only set at the defaults but it's there for the future.

With the Neuron built we turn to look at the paths between them, these can be defined as a start point (an input or group neuron) and an end point (a group or an output neuron) and a weight which the value being sent is multiplied by.

```
class PathSpec {
    has Str $.input;
    has Str $.output;
    has Rat() $.weight;
    method Str() { "{$.input}:{$.weight}:{$.output}" }
    method gist() { "{$.input} ==x{$.weight}==> {$.output}" }
    method COERCE( Str:D $str --> PathSpec:D ) {
        my ( $input, $weight, $output ) = $str.split(":");
        PathSpec.new( :$input, :$output, :$weight );
    }
}
```

The PathSpec class covers all this including the ability to transform them too or from Strings.

Finally we have the Brain (collection of Neurons and Paths).

```
class Brain {
    my $killScheduler;
    my $pathScheduler;

    has Supplier $.inputStream;
    has Supplier $!killStream;
    has @.watch-list;
    has @.outputs;
```

A Brain has an `$,inputStream` with all the input data and the `$!killStream` that will be connected to the `$!die` inputs on each Neuron in the Brain. The `@.watch-list` and `@.outputs` arrays contains the Promises from each `start` method and the combined output values from each `OutputNeuron`. We also defined 2 class level attributes, schedulers that will be shared between all the running brains. This is to stop the default scheduler from being overwhelmed.

```
    method kill() {
        $!killStream.emit( True );
    }

    submethod BUILD( :$!inputStream, :$!killStream, :@!watch-list, :@!outputs ) {}

    submethod DESTROY { self.kill(); }
```

A few methods to help building an tearing down brains and then we move to the make method. You can either give it a `gene` string a list of `PathSpec` strings joined by commas or a list of PathSpec Objects. 

```
    multi method make( Brain:U: Str :$gene!, :$inputStream, Bool :$scream) {
        my @paths = $gene.split(",").map( -> $g { my PathSpec(Str) $p = $g; $p });
        return Brain.make( :@paths, :$inputStream, :$scream );
    }

    multi method make( Brain:U: :@paths! is copy, :$inputStream = Supplier::Preserving.new(), Bool :$scream ) {
        my (@inputs, @outputs, @groups);

        my $gene = @paths.join(",");

        $pathScheduler //= ThreadPoolScheduler.new();
        $killScheduler //= ThreadPoolScheduler.new();
        my $killStream = Supplier.new();
        my $killSupply = $killStream.Supply();
        $killSupply.schedule-on($killScheduler);
```

Create the kill and path schedulers if they don't already exist, and an internal killstream that will be assigned to the private variable and the killSupply to pass to the Neurons.

```
        my @combined;
        repeat {
            my $ps = @paths.shift;
            for (@paths) -> $check is rw {
                if ( $ps.input ~~ $check.input && $ps.output ~~ $check.output ) {
                    $check = PathSpec.new(
                        :input($ps.input),
                        :output($ps.output),
                        :weight($ps.weight + $check.weight)
                    );
                    $ps = Nil;
                    last;
                }
            }
            @combined.push($ps) if $ps.defined;
        } while @paths;

        @paths = @combined;
```

With randomly generated genes we may end up with multiple connections between Neurons, this code combines them into single paths. 

```
        for (@paths) -> $p {
            given $p.input {
                when m/^i/ { @inputs.push($_) }
                when m/^g/ { @groups.push($_) }
            }
            given $p.output {
                when m/^g/ { @groups.push($_) }
                when m/^o/ { @outputs.push($_) }
            }
        }
        @inputs .= unique;
        @outputs .= unique;
        @groups .= unique;

        my $inputSupply = $inputStream.Supply();

        my %nodes;
        my @final-outputs;
        for ( @inputs ) -> $id {
            %nodes{$id} = InputNeuron.new( :$gene, :$id, :die($killStream.Supply()), :$scream );
            %nodes{$id}.attach-input($inputSupply);
        }
        for ( @outputs ) -> $id {
            %nodes{$id} = OutputNeuron.new( :$gene, :$id, :die($killStream.Supply()), :$scream );
            @final-outputs.push( %nodes{$id} );
        }
        for ( @groups ) -> $id {
            %nodes{$id} = TanHGroupNeuron.new( :$gene, :$id, :die($killStream.Supply()), :$scream );
        }
        for ( @paths ) -> $ps {
            my $path = Supplier.new();
            %nodes{$ps.input}.attach-output($path);
            %nodes{$ps.output}.attach-input($path.Supply
                                            .map( -> ($i,$v) { ($i, $v * $ps.weight) })
                                            .throttle(1, 0.5)
                                            .schedule-on($pathScheduler)
                                           );
        }
        my @watch-list = %nodes.values.map( *.start() ).grep( *.defined ).list;

        return Brain.new( :$inputStream, :@watch-list, outputs => @final-outputs, :$killStream );
    }
}

```

Then we create our Neurons and join them up based on the paths passed in. The paths are created using `map` for the weighting and some throttling to control feedback loops. Finally we pass these to the pathScheduler to ensure kill messages and other async process can still run safely.

Thinky was really happy with this system, for 1000 brains his 16 core machine handled just fine. With 5000 things got a bit crazy but still ran and generally he didn't think he'd need that much. All in all he was really happy with them... now he just had to remember why he'd made them. And work out how to train them and mutate them. But thatr was a job for another day (hopefully this advent calendar but this is very much a work in process).

I (I mean Thinky) hopes to get some of this released as a module soon if people are interested. 