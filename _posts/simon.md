---
layout: post
title: Everything you always wanted to know about Simon's algorithm * But were afraid to ask 
---

---
post_title: 'Simon's algorithm in Q#'
layout: post
published: false
---

# Recap: 1994, the year in which it all began

1994 was an eventful year. Some folks held their breath watching a white Ford Bronco in a high-speed car chase. Others enjoyed the new Euro tunnel feature that facilitated travel between France and England. A movie educated us about mouthwatering Big Kahuna burgers and the world-famous Jack Rabbit Slims Twist Contest. The world of figure scating saw the national champion stripped of her title. And in the world of animated sitcoms, America was introduced to two delightful teenagers who entertained us with their hilarious exploits. Moreover, 1994 is considered by many as the year when quantum computing went into warp speed after being considered a mere curiosity in the years prior. 

While some encouraging results about this new model of computation were already known by then, such as the Bernstein-Vazirani algorithm for Fourier sampling which---upon applying a nifty recursive construction---can be shown to give an advantage over classical computing that is bigger than any polynomial, the opening bell for age of quantum computing was really rung at FOCS 1994 which featured two seminal papers introducing Shor's quantum algorithms for solving discrete logarithms and for factoring of integers and Simon's quantum algorithm for figuring out bit-masks that describe the collisions of certain 2-to-1 functions. 

Due to its impacts on cryptography---after all, it shatters many of the primitives on which currently deployed public key cryptography relies---the former quantum algorithm is often mentioned as a poster child for quantum computing. Simon's algorithm has received quite a bit less attention, is however no less important nor less interesting. 

<b>TLDR:</b> This post deals with Simon's algorithm and shows how to implement it in Q#. We also managed to track down its creator Dan Simon, who was successfully hiding for 24 years in a Microsoft building until we found him using the corporate directory, and finally we share an anecdote about the two FOCS'94 quantum papers that might be new to some. 

# Simon's algorithm in Q# and C#

![Simon's algorithm](https://upload.wikimedia.org/wikipedia/commons/4/40/AltTeleport.jpg)
<p align="center">from <i>Quantum Computation and Quantum Information</i> by Nielsen and Chuang</p>

[Simon's algorithm](https://ref)

```C#
    operation SimonsAlgorithm (N : Int, Uf : ((Qubit[], Qubit[]) => ())) : Int[]
    {
        body
        {
            mutable resultBitmask = new Int[N];

            // allocate 2N qubits: N for the input register, N for the output register. 
            using (qs = Qubit[2*N])
            {
                // split allocated qubits into input register and output register
                let xs = qs[0..N-1];
                let ys = qs[N..2*N-1];

                // prepare qubits in the right state
                ApplyToEachA(H, xs);

                // apply the oracle
                Uf(xs, ys);

                // apply Hadamard to each qubit of the input register
                ApplyToEach(H, xs);

                // measure all qubits of the input register;
                // the result of each measurement is converted to a 0/1 value
                for (indexBitmask in 0..N-1)
                {
                    if (M(xs[indexBitmask]) == One)
                    {
                        set resultBitmask[indexBitmask] = 1;
                    }
                }

                // before releasing the qubits make sure they are all in the |0⟩ state
                ResetAll(qs);
            }

            return resultBitmask;
        }
    }
}


public static IEnumerable<object[]> GetInstances()
        {
            var assembly = System.Reflection.Assembly.GetExecutingAssembly();
            string resourceName = @"Quantum.Kata.SimonsAlgorithm.Instances.json";
            using (var stream = assembly.GetManifestResourceStream(resourceName)) {
                if (stream == null) {
                    var res = String.Join(", ", assembly.GetManifestResourceNames());
                    throw new Exception($"Resource {resourceName} not found in {assembly.FullName}. Valid resources are: {res}.");
                }
                using (var reader = new StreamReader(stream)) {
                    foreach (var instance in JsonSerializer.Create().Deserialize<List<Instance>>(new JsonTextReader(reader)))
                    {
                        yield return new object[] { instance };
                    }
                }
            }
        }

```Json
{
  "instance": 13,
  "transformation": [
    [ 0, 1, 1, 0 ],
    [ 1, 0, 0, 0 ],
    [ 0, 1, 0, 1 ]
  ],
  "kernel": [ 0, 1, 1, 1 ]
},
```

```C#
    public class GaussianElimination
    {
        /// <summary>
        /// Finds all vectors v for which vM = 0, where M is an argument matrix.
        /// If rank of matrix is less then minimalRank exception is raised to protect from too big kernel.
        /// </summary>
        /// <param name="matrix">linear transformation</param>
        /// <param name="minimalRank">required minimal rank</param>
        /// <returns></returns>
        public static List<BooleanVector> GetKernel(BooleanMatrix matrix, int minimalRank)
        {
            matrix = new BooleanMatrix(matrix);

            var columnPivot = new List<int?>();
            var row = 0;

            for (var column = 0; column < matrix.Width; column++)
            {
                var foundPivot = FindPivotAndSwapRows(matrix, row, column);
                if (foundPivot)
                {
                    ReduceRows(matrix, row, column);
                    columnPivot.Add(row);
                    row++;
                }
                else
                {
                    columnPivot.Add(null);
                }
            }

            if (row < minimalRank)
                throw new InvalidOperationException("Matrix doesn't have sufficient rank");

            return FindSolution(matrix, columnPivot, 0).Select(list => new BooleanVector(Enumerable.Reverse(list))).ToList();
        }
```

```C#
[Theory]
        [MemberData(nameof(GetInstances))]
        public void Test(Instance instance)
        {
            var sim = new OracleCounterSimulator();
            
            var len = instance.Kernel.Count;
            var saver = new List<QArray<long>>();

            for (int i = 0; i < len * 4; ++i)
            {
                var (vector, uf) = cs_helper.Run(sim, len, instance.ExtendedTransformation).Result;
                Assert.Equal(1, sim.GetOperationCount(uf));
                saver.Add(vector);
            }

            var matrix = new BooleanMatrix(saver);
            var kernel = matrix.GetKernel();

            Assert.Equal(instance.Kernel.Contains(true) ? 2 : 1, kernel.Count);
            Assert.Contains(instance.Kernel, kernel);
        }
```

# The 2018 Microsoft Hackathon team and tracking down Dan Simon

![The brave Hackathon team for Simon's kata](https://upload.wikimedia.org/wikipedia/commons/4/40/AltTeleport.jpg)
<p align="center">from <i>Quantum Computation and Quantum Information</i> by Nielsen and Chuang</p>


<img src="/pictures/simon.png" width="400px" />
