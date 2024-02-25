logo
Generic badge made-for-pascal

Noe is a framework to build neural networks (and hence, the name — noe (뇌): brain: 🧠) in pure object pascal. Yes, pascal, so you will have readable codes and pretty fast compiled executable binary. Some of its key features:

Automatic gradient computation
Creation of arbitrary rank tensor (a.k.a. multidimensional array) based on numerik library, that supports numpy-style broadcasting and is accelerated with OpenBLAS for the underlying heavy-lifting
(Optional) interface with GNU plot for plotting
Please note that although computation accelerator is applied, for the time being, noe is aimed for pedagogical purpose. If you want to create neural network in production with optimum speed, there are of course other choices.

Installation
Noe requires numerik, so you should install it first. Refer to numerik installation guide.
In lazarus, open "noe.source.lpk" package inside "pkg" directory. Open the package, compile, and add to project. Alternatively, you may also just include the "src" directory to the unit search path.
High-level neural network API
With automatic differentiation, it is possible to make of neural networks in various degree of abstraction. You can control the flow of of the network, even design a custom fancy loss function. For the high level API, there are several implementation of neural network layers, optimizers, along with TNNModel class helper, so you can prototype your network quickly.

program iris_classification;

{$mode objfpc}{$H+}

uses
  SysUtils, DateUtils, multiarray, numerik,
  noe2, noe2.optimizer, noe2.neuralnet;

var
  Dataset, X, Y, YBin, YPred, Loss: TTensor;
  model: TNNModel;
  opt: TOptAdam;
  i: integer;
  t: TDateTime;

begin
  Dataset := ReadCSV('iris.csv');

  X := Dataset[[ _ALL_, Range(0, 4) ]]; // Get all rows and first four columns
  Y := Dataset[[ _ALL_, 4 ]]; // Get all rows and a column with index 4
  YBin := BinarizeLabel(Y); // Transform labels into one-hot vectors

  model := TNNModel.Create;
  model.AddLayer(TLayerDense.Create(4, 30));
  model.AddLayer(TLayerReLU.Create());
  model.AddLayer(TLayerDense.Create(30, 3));
  model.AddLayer(TLayerSoftmax.Create(1));

  opt := TOptAdam.Create(model.Params); // Adam optimizer
  opt.LearningRate := 0.01;

  t := Now;
  for i := 0 to 100 do
  begin
    YPred := model.Eval(X);
    Loss := CrossEntropy(YPred, YBin);
    Loss.Backward();
    opt.Step;

    if i mod 10 = 0 then
      WriteLn('Loss at iteration ', i, ': ', Loss.Data.Get(0) : 5 : 2);
  end;

  WriteLn('Training completed in ', MilliSecondsBetween(Now, t), ' ms');
  WriteLn('Training accuracy: ', Mean(ArgMax(YPred.Data, 1, True)).Item : 5 : 2);
  WriteLn('Press enter to exit'); ReadLn;

  model.Free;
  opt.Free;
end.  
logo
Aaaand... you are good to go. More layers are coming soon (including convolutional layers).

Touching the bare metal: Write your own math
Noe is hackable. If you want more control, you can skip TNNModel and TLayer creation and define your own model from scratch. It is easy and straightforward, like how normal people do math. No random cryptic symbols. Following is an example of noe usage to solve XOR problem.

program xor_example;

uses
  multiarray, numerik, noe;

var
  X, y, yPred, Loss: TTensor;
  W1, W2, b1, b2: TTensor; // Weights and biases
  LearningRate: Single;
  i: integer;

begin
  Randomize;

  X := CreateMultiArray([0, 0,
                         0, 1,
                         1, 0,
                         1, 1]).Reshape([4, 2]);
  y := CreateMultiArray([0, 1, 1, 0]).Reshape([4, 1]);

  W1 := Random([2, 5]); // Input to hidden
  W2 := Random([5, 1]); // Hidden to output
  W1.RequiresGrad := True;
  W2.RequiresGrad := True;

  b1 := Zeros([5]);
  b2 := Zeros([1]);
  b1.RequiresGrad := True;
  b2.RequiresGrad := True;

  LearningRate := 0.01;
  for i := 0 to 2000 do
  begin
    yPred := (ReLu(X.Matmul(W1) + b1)).Matmul(W2) + b2; // Prediction
    Loss := Mean(Sqr(yPred - y)); // MSE error

    W1.ZeroGrad;
    W2.ZeroGrad;
    b1.ZeroGrad;
    b2.ZeroGrad;

    Loss.Backward(); // Backpropagate the error and compute gradients
    
    { Update the parameters }
    W1.Data := W1.Data - LearningRate * W1.Grad;
    W2.Data := W2.Data - LearningRate * W2.Grad;
    b1.Data := b1.Data - LearningRate * b1.Grad;
    b2.Data := b2.Data - LearningRate * b2.Grad;

    if i mod 50 = 0 then
      WriteLn('Loss at iteration ', i, ': ', Loss.Data.Get(0) : 5 : 2);
  end;

  WriteLn('Prediction:');
  PrintTensor(YPred);

  Write('Press enter to exit'); ReadLn;
end.  
logo
That said, you could have even defined your own custom layers and optimizers 🤘. Really. Even noe's layer implementations are pretty verbose and straightfowrward. Check the source code yourself whenever you have free time.

You can also compute the loss function derivative with respect to all parameters to obtain the gradients... by your hands... But just stop there. Stop hurting yourself. Use more autograd.

See the wiki for more documentation. Please note that this framework is developed and heavily tested using fpc 3.0.4, with object pascal syntax mode, on a windows machine. Portability is not really my first concern right now, but any helps are sincerely welcome. See CONTRIBUTING.md.
